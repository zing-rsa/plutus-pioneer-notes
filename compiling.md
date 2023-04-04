
# Compiling untyped scripts

- untyped scripts are scripts which take in parameters(datum, redeemer) that are of type `BuiltInData`
    - lowest level of param, easy to compile
- follow pattern of `BuiltInData -> BuiltInData -> BuiltInData -> ()`
```
mkGiftValidator :: BuiltInData -> BuiltInData -> BuiltInData -> ()
mkGiftValidator _ _ _ = ()

validator :: PlutusV2.Validator
validator = PlutusV2.mkValidatorScript $$(PlutusTx.compile [|| mkGiftValidator ||])
```

# Compiling untyped scripts

- typed scripts are scripts that have params(datum, redeemer) which can be arbitrary types
- wrapping is required the turn these scripts to scripts of type `BuiltInData -> BuiltInData -> BuiltInData -> ()`
```
                Datum  Redeemer        ScriptContext
mk42Validator :: () -> Integer -> PlutusV2.ScriptContext -> Bool
mk42Validator _ r _ = traceIfFalse "expected 42" $ r == 42
{-# INLINABLE mk42Validator #-}

validator :: PlutusV2.Validator
validator = PlutusV2.mkValidatorScript $$(PlutusTx.compile [|| wrapValidator mk42Validator ||])
                                                                 ^^^^^^
                                                 this is a helper function to do wrapping
```
>wrapValidator sneak peek
>```
> {-# INLINABLE wrapValidator #-}
> wrapValidator :: ( UnsafeFromData a, UnsafeFromData b)
>               => (a -> b -> ScriptContext -> Bool)
>               -> (BuiltinData -> BuiltinData -> BuiltinData -> ())
> wrapValidator f a b ctx =
>   check $ f
>       (unsafeFromBuiltinData a)
>       (unsafeFromBuiltinData b)
>       (unsafeFromBuiltinData ctx)
>```

# Compiling parameterized scripts
- parameterized scripts need some more steps to compile because parameters need to be compiled into the script
- this changes the signature of the validator
- also presents a challenge because the parameter value is not known at compile time

Example:
```
data VestingParams = VestingParams
    { beneficiary :: PubKeyHash, deadline :: POSIXTime}

{-# INLINABLE mkParameterizedVestingValidator #-}
mkParameterizedVestingValidator :: VestingParams -> () -> () -> ScriptContext -> Bool
mkParameterizedVestingValidator params () () ctx = True
```

#### Step 1 - update the wrapper:
```
{-# INLINABLE  mkWrappedParameterizedVestingValidator #-}
                                        note the new param
                                           vvv
mkWrappedParameterizedVestingValidator :: VestingParams -> BuiltinData -> BuiltinData -> BuiltinData -> ()
mkWrappedParameterizedVestingValidator = wrapValidator . mkParameterizedVestingValidator
                                                     ^^
                                           dot syntax required here
```

#### Step 2 - update the validator 
```
                new param
                  vvv
validator :: VestingParams -> Validator
validator p = mkValidatorScript $$(compile [|| mkWrappedParameterizedVestingValidator p ||])
         ^^                                                                           ^^
      add p so types are correct                                                add p so types are correct
```

#### problem!:
there is a problem with the above code, even though the haskell compiler won't thing there is. 
- the `[|| .. ||]` syntax is a template haskell function that inserts compiled code as if it was written in place, called `splicing`
- when template haskell tries to `splice` a piece of code, it must know what the code is at compile time(otherwise it won't know what to generate)
- the code above won't know what `p` is at compile time because it is a param that is only know when it is passed to the script at runtime

### solution:
- since we are able to compile the validator script, if we could only compile `p` then we could apply the validator script to `p`
- because `p` will always be only data(no functions), it can be compiled at runtime using a type called `liftCode`
- once we have a `liftCode` of `p` at runtime, we can use a function called `'applyCode'` to apply the validator script to the `p` `liftCode`  

note: `Lift` can only be applied to data types, not function type   

`LiftCode` signature: `LiftCode :: a -> CompiledCodeIn`

#### step 1
Turn our VestingParams object into a Lift
```
data VestingParams = VestingParams
    { beneficiary :: PubKeyHash
    , deadline    :: POSIXTime
    }

makeLift ''VestingParams
 ^^^^^^^^^^^^^^^^
```

#### step 2
Update our validator to apply the lift to the validator using `applyCode`
```
validator :: VestingParams -> Validator
validator params = mkValidatorScript ($$(compile [|| mkWrappedParameterizedVestingValidator ||]) `applyCode` liftCode params)
                                                                                                    ^^^^^^^^^^^^^^^^^^^^^^
                                                              applying the compiled script to the params liftcode that is compiled at runtime
```

#### Notes: 
- can have multiple params, in which case you need to chain several `applyCodes` to apply the params one after the other, and would need to lift those params as well