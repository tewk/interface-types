# Credit Card Processing

## Preamble
This note is part of a series of notes that gives complete end-to-end examples
of using Interface Types to export and import functionality.

## Introduction
Many APIs involve passing complex structured information to the function -- for
example the information associated with paying by credit card includes the
credit card data: name, number, expiration date and so on.

Credit card processing is an example of a compelling use case for WebAssembly:
creating a client library around a service functionality. In this note we are
focused on how the client library can be made specified and used using Interface
Types.

## Defining Types

The Interface Types type language includes a notation for declaring types using
_algebraic type definitions_.

We can define a `cc` type -- used to represent the details of a credit card --
using two `datatype` statements:

```wasm
(@interface datatype $cc 
  (record
    (field "ccNo" u64)
    (field "name" string)
    (field "expires" (type $ccExpiry))
    (field "ccv" u16)
  )
)

(@interface datatype $ccExpiry
  (record
    (field "mon" u8)
    (field "year" u16)
  )
)
```

The second language feature we would like to introduce is the concept of a
_service type_. A service type defines what can be done with a given entity: it
consists of a set of named method definitions.[^Service type is analogous to the
interface type in webIDL.]

In an actual payment processing API there is likely to be the equivalent of a
_payment session_. A payment session can be viewed as a resource which is
associated with a set of operations or methods on that resource. One of the
methods against a session will be the one that actually initiates payment; we
can define this using the definition:

```wasm
(@interface service $ccService
  (method "payWithCard"
    (func (param "card" (type $cc))
          (param "amount" u64)
          (param "update" (ref $statusCallback))
          (result u64))))
```

>Note that we have only shown one method in the `$ccService` service; in most
>cases there will be more.

The `payWithCard` method takes a credit card record, a transaction amount
(assumed in this case to be an unsigned number of cents) and a callback --
`"update"`. The callback is a function argument that is used to signal to the
caller the status of the transaction. The idea is that the `update` function
will be invoked when the payment has completed -- successfully or otherwise.

The type of `update` is given in the statement:

```wasm
(@interface datatype $statusCallback
  (func (param "transaction" u64)
        (param "status" (type $processStatuss))))
```

The status of the transaction is reported as a `$processState` value. There are
two variants of this type: representing whether the payment has completed
successfully, or it has `$failed` -- in which case there is a message to
display.

The definition of the `$processStatus` type is an example of an algebraic
datatype. The `datatype` statement declares that there are two variants of the
type -- labeled `$completed` and `$failed`. The former is effectively an
enumerated symbol, the latter is a record with the field `"reason"` signaling
the cause of the failure:

```wasm
(@interface datatype $processStatus
  (oneof
    (variant $completed)
    (variant $failed (type $reason))))

(@interface datatype $reason
  (record
    (field "reason" string)))
```

Such polymorphism is quite common; and it is not always restricted to such super
simplistic cases as here. In fact, if we wanted to take internationalization
into account, then the `reason` field should itself have a type that reflects
the different ways of encoding international messages.

>Note: although the Interface Types language has algebraic data type
>definitions, the schema _does not_ allow recursive data types to be
>defined. This is in keeping with the overall mission of Interface Types.

## Credit Card Processing

Actually exporting a service method is done with a variant of the `@interface`
element that defines methods.  We can see this in the export adapter:

```wasm
(@interface method (export "payWithCard")
  (param $session (type $ccService))
  (param $card (type $cc))
  (param $amount u64)
  (param $update (ref $statusCallback))
  (result u64)

  local.get $card
  unpack (type $cc)
  let (local ($ccNo u64)
             ($name string)
             ($expires (type $expiry))
             ($ccv u16))
    local.get $ccNo   ;; access ccNo
    u64-to-i64

    local.get $name
    string.size
    call-export "malloc"
    deferred (i32)
      call-export "freex"
    end
    string-to-memory "memx"
  
    local.get $expires
    unpack (type $ccExpiry)
    let (local ($mon u8) (local $year u16))
      local.get $mon
      local.get $year
    end
    local.get $ccv
  
    local.get $amount
    u64-to-i64
  
    local.get $session
    resource-to-eqref
    
    local.get $update
    func.ref $ccReportStatus
    func.bind (func (param u64) (param (type $processStatus)))
    
    call-export "payWithCard_impl"
    i32-to-u64
  end
)
```

There are several moving pieces to this code; it is worth examining them
individually:

1. The `$session` parameter -- which is asserted to be a `$ccService` -- is
   passed as the first parameter. This represents a style of implementing
   services where each method is implemented as a separate function.

1. The fragment:

   ```
   unpack (type $cc)
   let (local ($ccNo u64)
             ($name string)
             ($expires (type $expiry))
             ($ccv u16))
   ```
   examines an argument -- whose type is a `$cc` -- and unpacks it into its
   component parts using an `unpack` instruction. 

   The `unpack` instruction takes a record and unpacks it into its constituent
   elements -- which are taken off the stack in this adapter with a `let`
   instruction.
  
1. the string name on the card is copied into linear memory using the fragment:

   ```
   local.get $name
   string.size
   call-export "malloc"
   deferred (i32)
     call-export "freex"
   end
   string-to-memory "memx"
   ```

   The memory allocated in this process will eventually be freed when it is no
   longer needed.
  
1. The callback function -- whose type is expressed as an Interface Type -- must
   be passed to the core WebAssembly implementation of
   `processCardImpl`. However, that function does not understand the additional
   semantics of Interface Types. So, we have, in effect, to _import_ the
   callback and provide an import adapter for it.
   
   This involves using the `$ccReportStatus` function as the wrapper.  The
   function reference passed to the `processCarImpl` function is a result of
   using `func.bind` on the combination of `$ccReportStatus` and the dynamically
   passed function:
   
   ```wasm
   local.get $update
   func.ref $ccReportStatus
   func.bind (func (param u64) (param (type $processStatus)))
   ```

1. The `$ccReportStatus` function is not a normal import adapter, nor is it a
   regular WebAssembly function. It's type reflects this: it involves both
   Interface Types and regular core WebAssembly types.
   
   `$ccReportStatus` is not intended to be invoked in a 'raw' form -- it is
   intended to be bound with an export adapter using the `func.bind` instruction
   from [Function References].
   
   On the other hand, `$ccReportStatus` does look like an import adapter:
   
   ```wasm
   (@interface func $ccReportStatus
     (param $callBk (ref $statusCallback))
     (param $txId i64)
     (param $status i32)
     (param $failString i32)
     (param $failStringLength i32)

     local.get $txId
     i64-to-u64

     block $fail
       local.get $status
       br_if $fail
       vary $completed (type $processStatus)
       local.get $callBk
       return_call_indirect
     end $fail

     local.get $failString
     local.get $failStringLength
     memory-to-string "memx"
     pack (type $reason)
     vary $failed (type $processStatus)
     local.get $callBk
     return_call_indirect
   )

   ```

1. As part of its role an an import adapter, `$ccReportStatus` has to lift a
   status code into the `$processStatus` type. That type is expressed as an
   algebraic type, specifically a combination of an enumeration and a record --
   representing the success and failure cases respectively.
   
1. Normal WebAssembly does not have the concept of discriminated union
   types. This means that there has to be a convention on how to signal which
   arm is implied. In this case we are assuming that the underlying callback
   uses three arguments: a flag that is `0` if the callback is reporting
   completion of the request and `1` if it is reporting failure. In the latter
   case, the second and third arguments represent the error string and its
   length.
   
   Converting a flag into an enumeration value (which is simply one of the
   variants of an algebraic type) is accomplished using a `vary` instruction:
   
```
    vary $completed (type $processStatus)
```

1. The logic to convert the combination of the three arguments into the single
   `$processStatus` value is in the fragment:
   
```
  block $fail
    local.get $status
    br_if $fail
    vary $completed (type $processStatus)
    local.get $callBk
    return_call_indirect
  end $fail

  local.get $failString
  local.get $failStringLength
  memory-to-string "memx"
  pack (type $reason)
  vary $failed (type $processStatus)
```

  The two `vary` instructions pick their appropriate variants from the algebraic
  type definition.
  
1. The `pack` instruction is the complement to the `unpack` instruction: it
   takes elements from the stack and uses them to construct a record value.
  
1. The call to the callback itself is accomplished using the
   `return_call_indirect` instruction. We can use a tail call because there is
   no further action required after the callback has been invoked:
   
```
  local.get $callBk
  return_call_indirect
```

  Note that `$callBk` is not a regular parameter or local variable for two
  reasons:
  
  1. It is actually an export adapter function -- corresponding to the callback constructed in the appropriate import adapter for `$ccProcessPayment`.
  
  1. It is bound to the function via the `func.bind` -- it is not passed as a
     parameter in a normal call.

Although reasonably long, this export adapter is quite straightforward: it
unpacks the credit card record (and the expiration month within that) and passes
the resulting information to an internally exported implementation function --
`"payWithCard_impl"`.


## Paying with a Credit Card

Accessing services -- such as a credit card payment service -- is a mirror of
the export story for providing services. In particular, there is a variant of
function call that reflects accessing methods in a service -- the `invoke`
adapter instruction.

Note that core WebAssembly does not have the concept of invoking a method on an
object -- it only has function call. Thus the import adapter for such an import
has to map the function call to an invokation of a method on a service.

Other than that, the other major tasks for our adapter are to construct the
credit card data and to arrange for the callback.

In this case, we assemble the credit card record from a memory-based block of
data passed to the core import.

This is reflected in the core WebAssembly function import statement:

```wasm
(func $payWithCard_ (import "" "payWithCard_")
  (param anyref i32 i64 func.ref)
  (result i64))
```

1. The first first argument is the session -- represented in core WebAssembly as an `anyref` value;
1. the second argument is a pointer to a memory region containing the credit card data;
1. the third argument is the transaction amount (in cents in US currency)
1. the last argument is a function reference to a callback function to invoke.

The return value is intended to represent a transaction id.

In order to access the service method, the credit card record must be read out
of memory and `pack`ed into a `cc` record. In addition the service is invoked
against the passed-in `eqref` using an `invoke` instruction:

```wasm
(@interface implement (import "" "payWithCard_")
  (param $session eqref)
  (param $cc i32)
  (param $amnt i64)
  (result i32)

  local.get $cc
  i64.load {offset #cc_ccNo}
  i64-to-u64

  local.get $cc
  i32.load {offset #cc_name_ptr}

  local.get $cc
  i32.load {offset #cc_name_len}
  memory-to-string "memi"
  
  local.get $cc
  i16.load_u {offset #cc_expires_mon}
  i32-to-u8

  local.get $cc
  i16.load_u {offset #cc_expires_year}
  i32-to-u16

  pack (type $ccExpiry)
  
  local.get $cc
  i16.load_u {offset #cc_ccv}
  i32-to-u16

  pack (type $cc)
  
  local.get $amnt
  i64-to-u64
  
  local.get $session
  eqref-to-service (type $ccService)
   
  invoke "payWithCard"
  enum-to-i32 boolean
)
```

The `eqref-to-service` instruction casts an `anyref` (specifically one that
supports equality -- an `eqref`)[^Quiz: Why does it need to be an `eqref`?] to
an entity that implements the `ccService` service type.

[Function References]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md
