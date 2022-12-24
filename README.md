# Contractum: RGB smart contract language

Repository is a prototyping for possible RGB contract language, which should
compile to RGB Schema data with additional validation code running on virtual
machine(s) supported by RGB (currently this will be AluVM)

Contractum differs from other smart contract programming languages in a fact
that as functional as Haskell and nearly as close to the bare metal as Rust at
the same time, filling in the space which was not accessible for the smart
contracts before:

![contractum-box-black](https://user-images.githubusercontent.com/372034/209446013-24638564-08ec-4eff-afbb-302f83e5b0df.png)

For a sample
code pls check [RGB20 fungible token](attempt3/rgb20.con)
contract, which should compile into RGB20 Schema. You may also see how code 
compilation into [AluVM] assembly [may look like](attempt3/rgb20.aluasm)
and check [PEG] [grammar of contractum language](attempt3/grammar.pest).

Here is an example of Contractum code[^highlight]:
```Haskell
types Did
   data PgpKey :: curve U8, key Bytes

schema DecentralizedIdentity
   -- This defines the atom of the contract state called `Identity` 
   -- which has data type `PgpKey`.
   -- The `owned` keyword means that there is always a party
   -- which owns the identity
   owned Identity :: Did.PgpKey

   owned IOYIssue :: Zk64
   -- `Zk64` means 64-bit unsigned integer hidden with zero-knowledge
   owned IOYTokens :: Zk64

   global IOYTicker :: String
   global IOYName :: String

   -- This says that to construct contract the user must provide
   -- information about exactly one identity and its IOY token
   genesis :: Identity, IOYTicker, IOYName

   -- Now let's define what a owner of identity can do,
   -- He can execute his rights by creating state transitions
   -- ("operation" on the state) of predefined forms, like
   op Revocation :: old Identity -> new Identity
   -- which does what it says: it revokes existing identity
   -- and creates a new one.
   
   -- This issues new IOY promises in tokenized form
   op Promise :: used IOYIssue -> given [IOYTokens]?, remaining IOYIssue?
      assert used == sum given + (remaining ?? 0)

   -- This transfers IOY tokens from one owner to another owner
   op Transfer :: spent {IOYTokens} -> received [IOYTokens]
      assert sum spent == sum received
   
interface FungibleToken:
   global Ticker -> String -- this is similar to schema definition; in fact
                           -- it is a requirement that the schema must provide
                           -- a global state of the String type and link it to
                           -- the "Ticker" name
   global Name -> String

   owned Inflation :: Zk64 -- pretty much the same applies to assigned state
   owned Asset :: Zk64

   op Issue :: Inflation -> [Asset]?, Inflation? -- and operations
   op Transfer :: {Asset} -> [Asset]

interface PgpIdentity
  owned Identity :: PgpKey
  exec Revocation :: old Identity -> new Identity

-- Specific schema state may use different naming, for instance because a schema
-- can define multiple assets with different names; in that case we will have
-- multiple interface implementations referencing different state.
implement FungibleToken for DecentralizedIdentity
   global Ticker := IOYTicker -- this creates a _binding_ of the state defined
                              -- in the schema (*IOYTicker* in this case) to the
                              -- interface 
   global Name := IOYName
   owned Inflation := IOYIssue
   owned Asset := IOYTokens
   op Issue := Promise
   op Transfer -- here we skip `:=` part since the interface operation name
               -- matches the name used in the schema. In such cases we can also
               -- skip the declaration at whole

implement PgpIdentity for DecentralizedIdentity
  -- we do not need to put anything here since schema state and operation names
  -- matches interface requirements and the compiler is able to guess the bindings
  
contract meSatoshiNakamoto implements DecentralizedIdentity
   set IOYTicker := "SATN"
   set IOYName := "Satoshi Promises"
   -- this defines a genesis state and assigns it to a single-use-seal
   assign orig Identity := (0xfac503c4641c3deda72a2d00bc9d6ff1094b15276c386efea403746a91436772, 1) 
                        -> PgpKey(0, 0x028730eeeec41802621d177507b086f390ae600ba3ca5e428b13913af4c2cd25b3)

transition iLostMyKey executes Revocation
   via meSatoshiNakamoto.orig -- specifies the single-use-seal we close to match requirements
                              -- on the valid operation execution conditions
   assign upd Identity := (~, 2) -- here we use txid of the bitcoin transaction which will be
                                 -- created to hold the commitment to this state transition, 
                                 -- called "single-use-seal witness". Since we can not know the
                                 -- txid upfront we use ~ to indicate the witness transaction id
                       -> PgpKey(0, 0x0219db0a4e0eb8cb833608c08d76b9b279ec44a851ab82cc6fd68a9b32624bfa8b)
   -- the above defines new state and assigns it to a single-use-seal
```

[AluVM]: https://www.aluvm.org
[PEG]: https://en.wikipedia.org/wiki/Parsing_expression_grammar 

[^highlight]: Here we use Haskell language highlighter, which does not recognizes 
Contractum keywords, but at least due to a similarity of Contractum to the 
Haskell languague provides some better visual recognizability
