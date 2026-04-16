# Haskell Numeric Conversions

`fromIntegral` is one of the most dangerous functions in Haskell. It silently
truncates, wraps, or loses precision with no indication at the call site. This
document describes safer alternatives using the
[unwitch](https://hackage.haskell.org/package/unwitch) library.

## The Problem with fromIntegral

```haskell
fromIntegral (maxBound :: Int) :: Int32
-- Silently wraps to -1 on 64-bit systems. No warning, no error.

fromIntegral (2^60 :: Int) :: Double
-- Silently loses precision. The result is approximate.

fromIntegral (-1 :: Int) :: Word8
-- Silently wraps to 255. Perfectly type-safe, completely wrong.
```

The type signature `(Integral a, Num b) => a -> b` tells you nothing about
whether the conversion is safe, lossy, or total nonsense.

## The unwitch Approach

unwitch provides one module per source type, with named conversion functions
that make safety properties visible at the call site:

```haskell
import qualified Unwitch.Convert.Int as Int
import qualified Unwitch.Convert.Int32 as Int32
import qualified Unwitch.Convert.CInt as CInt
import qualified Unwitch.Convert.Word8 as Word8
```

Total (safe) conversions return the target type directly. Partial (potentially
failing) conversions return `Maybe` or `Either`.

## Total Conversions (never fail)

These are always safe — the target type can represent every value of the source:

| From -> To | Function | Notes |
|---|---|---|
| `Int32 -> CInt` | `Int32.toCInt` | CInt is newtype over Int32 |
| `Int32 -> Double` | `Int32.toDouble` | All Int32 fit in Double exactly |
| `CInt -> Int32` | `CInt.toInt32` | CInt is newtype over Int32 |
| `CInt -> Int` | `CInt.toInt` | Widening, always safe |
| `CInt -> Double` | `CInt.toDouble` | All Int32 fit in Double exactly |
| `Word8 -> CInt` | `Word8.toCInt` | Widening |

## Partial Conversions (can fail)

These may fail because the target type cannot represent all values of the source:

| From -> To | Function | Return | When it fails |
|---|---|---|---|
| `Int -> CInt` | `Int.toCInt` | `Maybe CInt` | Int outside Int32 range |
| `Int -> Int32` | `Int.toInt32` | `Maybe Int32` | Int outside Int32 range |
| `Int -> Double` | `Int.toDouble` | `Either Overflows Double` | Int > 2^53 (precision loss) |
| `Int -> Word8` | `Int.toWord8` | `Maybe Word8` | Int < 0 or > 255 |
| `CInt -> Int16` | `CInt.toInt16` | `Maybe Int16` | CInt outside Int16 range |

## Handling Partial Conversions

**Never use `error` for failed conversions.** Prefer these patterns:

### 1. Use total conversions where possible

Choose types that allow total conversion. If a value is always small, store it
as `CInt` instead of `Int`:

```haskell
hexSize :: CInt    -- not Int, since consumers need CInt
hexSize = 80
-- Now CInt.toDouble hexSize is total, no Maybe needed
```

### 2. Use floor/round targeting the right type

Instead of `floor :: Double -> Int` then `Int.toCInt`, target the output type
directly:

```haskell
floor someDouble :: CInt    -- floor targets CInt directly via Integral instance
round someDouble :: CInt    -- same for round
```

### 3. Default value for known-safe conversions

When you know values are small (grid coordinates, UI indices, counters):

```haskell
toCInt' :: Int -> CInt
toCInt' = maybe 0 id . Int.toCInt  -- 0 default unreachable for small values
```

### 4. Clamp for genuinely narrowing conversions

For rendering or hardware interfaces where out-of-range should clamp, not crash:

```haskell
cintToInt16Clamp :: CInt -> Int16
cintToInt16Clamp c = case CInt.toInt16 c of
  Just i  -> i
  Nothing -> if CInt.toInt c > 0 then maxBound else minBound
```

### 5. Restructure loops to avoid conversion

Instead of converting a loop variable:

```haskell
-- Bad: forM_ [1..20 :: Int] $ \i -> setAlpha (fromMaybe (error "...") (Int.toWord8 (i * 11)))
-- Good: forM_ [11, 22 .. 220 :: Word8] $ \alpha -> setAlpha alpha
```

## Checking Available Conversions

Browse the docs for a specific source type:

```bash
w3m -dump "https://hackage.haskell.org/package/unwitch/docs/Unwitch-Convert-Int.html"
```

Or search Hoogle:

```bash
curl -s "https://hoogle.haskell.org/?mode=json&hoogle=Unwitch.Convert.Int"
```
