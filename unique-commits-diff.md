# Unique Commit File Changes

These are the file changes introduced by the **8 commits** that exist in `44e0d3b9` but not in upstream `o1-labs/proof-systems` tag `0.5.0`.

**Common base:** `fd7c19d4` (feat(mina-signer): serialization to hex for secret key)

**Summary:** 7 files changed, 58 insertions(+), 14 deletions(-)

---

## `signer/src/keypair.rs` — +37 / -6

### Refactor `from_hex` to delegate to new `from_bytes`

```diff
-    /// Deserialize a keypair from secret key hex
-    ///
-    /// # Errors
-    ///
-    /// Will give error if `hex` string does not match certain requirements.
-    pub fn from_hex(secret_hex: &str) -> Result<Self> {
-        let mut bytes: Vec<u8> = hex::decode(secret_hex).map_err(|_| KeypairError::SecretKeyHex)?;
-        bytes.reverse(); // mina scalars hex format is in big-endian order
-
-        let secret = ScalarField::from_bytes(&bytes).map_err(|_| KeypairError::SecretKeyBytes)?;
+    /// Deserialize a keypair from secret key bytes
+    pub fn from_bytes(bytes: &[u8]) -> Result<Self> {
+        let secret = ScalarField::from_bytes(bytes).map_err(|_| KeypairError::SecretKeyBytes)?;
         let public: CurvePoint = CurvePoint::prime_subgroup_generator()
             .mul(secret)
             .into_affine();
         Ok(Keypair::from_parts_unsafe(secret, public))
     }
+
+    /// Deserialize a keypair from secret key hex
+    ///
+    /// # Errors
+    ///
+    /// Will give error if `hex` string does not match certain requirements.
+    pub fn from_hex(secret_hex: &str) -> Result<Self> {
+        let mut bytes: Vec<u8> = hex::decode(secret_hex).map_err(|_| KeypairError::SecretKeyHex)?;
+        bytes.reverse(); // mina scalars hex format is in big-endian order
+        Self::from_bytes(&bytes)
+    }
```

### Add `to_hex` serialization method

```diff
+    /// Serialize a keypair as a hex of secret key
+    pub fn to_hex(&self) -> String {
+        let mut bytes = self.secret.scalar().to_bytes();
+        bytes.reverse();
+        hex::encode(&bytes)
+    }
```

### Add `secret_multiply_with_curve_point` method

```diff
+    /// Performs scalar multiplication with the secret key
+    pub fn secret_multiply_with_curve_point(&self, multiplicand: CurvePoint) -> CurvePoint {
+        multiplicand.mul(self.secret.clone().into_scalar()).into_affine()
+    }
```

### Add `to_hex` round-trip test

```diff
+    #[test]
+    fn to_hex() {
+        let secret_hex = "164244176fddb5d769b7de2027469d027ad428fadcc0c02396e6280142efb718";
+        let keypair = Keypair::from_hex(secret_hex).expect("failed to decode keypair secret key");
+        assert_eq!(keypair.to_hex(), secret_hex);
+    }
```

---

## `signer/src/pubkey.rs` — +7 / -1

### Add `PartialOrd`, `Ord` and custom `Debug` for `CompressedPubKey`

```diff
-#[derive(Clone, Debug, PartialEq, Eq)]
+#[derive(Clone, PartialEq, Eq, PartialOrd, Ord)]
 pub struct CompressedPubKey {
     pub x: BaseField,
     pub is_odd: bool,
 }

+impl fmt::Debug for CompressedPubKey {
+    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
+        f.write_fmt(format_args!("{}", self.into_address()))
+    }
+}
```

---

## `signer/src/signature.rs` — +10 / +0

### Add `Signature::dummy()` method

```diff
+    /// Dummy signature
+    pub fn dummy() -> Self {
+        use ark_ff::One;
+
+        Self {
+            rx: BaseField::one(),
+            s: ScalarField::one(),
+        }
+    }
```

---

## `kimchi/src/prover.rs` — +3 / -3

### Refactor `create_recursive` to accept explicit `rng` parameter

```diff
+use rand_core::{CryptoRng, RngCore};

     /// non-recursive case
     pub fn create(
         ...
+        &mut rand::rngs::OsRng,   // call site updated
     )

     pub fn create_recursive(
         index: &ProverIndex<G>,
         prev_challenges: Vec<RecursionChallenge<G>>,
         blinders: Option<[Option<PolyComm<G::ScalarField>>; COLUMNS]>,
+        rng: &mut (impl RngCore + CryptoRng),
     ) -> Result<Self> {
-        // TODO: rng should be passed as arg
-        let rng = &mut rand::rngs::OsRng;
```

---

## `kimchi/src/tests/framework.rs` — +1 / 0

### Update test call site for new `create_recursive` signature

```diff
         ProverProof::create_recursive::<EFqSponge, EFrSponge>(
             &prover,
             self.0.recursion,
             None,
+            &mut rand::rngs::OsRng,
         )
```

---

## `circuit-construction/src/prover.rs` — +3 / 0

### Update `prove` function signature and call site

```diff
+use rand_core::{CryptoRng, RngCore};

 pub fn prove<G, H, EFqSponge, EFrSponge>(
     ...
     mut main: H,
+    rng: &mut (impl RngCore + CryptoRng),
 ) -> ProverProof<G> {
     ...
     ProverProof::create_recursive(
         index,
         vec![],
         Some(blinders),
+        rng,
     )
```

---

## `circuit-construction/src/tests/example_proof.rs` — +1 / 0

### Pass `rng` to updated `prove` call

```diff
     prove::<Pallas, _, BaseSponge, ScalarSponge>(
         ...
         |sys, p| circuit::<Fp, Pallas, _>(&proof_system_constants, Some(&witness), sys, p),
+        &mut rand::rngs::OsRng,
     );
```

---

## Summary

All changes are confined to the **`signer/`** module and its downstream call sites, covering three areas:

1. **`signer` module** — new APIs: `from_bytes`, `to_hex`, `secret_multiply_with_curve_point`, `Signature::dummy`, ordering traits and custom `Debug` for `CompressedPubKey`
2. **`kimchi` prover** — `rng` refactored from internal hardcode to explicit parameter
3. **Test / call-site adapters** — updated to match new function signatures
