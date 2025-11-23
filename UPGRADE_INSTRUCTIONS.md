# BondingCurve Upgrade Required

## ⚠️ IMPORTANT ISSUE

The deployed BondingCurve contract at `0x2cA9DBEB3D6931c07B0819D446Ee3276F4154cc8` **does NOT** have the `setGraduationManager()` function.

This is because it was deployed BEFORE we added the DEX graduation feature.

## 🔧 Solution Options

### Option 1: Redeploy BondingCurve (RECOMMENDED)

**Pros:**
- Clean deployment with all new features
- No migration needed
- Fresh start

**Cons:**
- All existing tokens stay on old curve
- Need to update TokenFactory to point to new curve
- Users need to know about new deployment

**Steps:**
1. Deploy new BondingCurve with graduation manager support
2. Deploy new TokenFactory pointing to new BondingCurve
3. Deploy DEXGraduationManager (already done)
4. Set graduation manager in new BondingCurve
5. Update frontend to use new addresses

### Option 2: Deploy Separate BondingCurve for New Tokens

**Pros:**
- Existing tokens continue to work
- Old tokens don't have DEX graduation (simple)
- New tokens get DEX graduation

**Cons:**
- Two BondingCurves to maintain
- Confusing for users
- Split liquidity

### Option 3: Manual Graduation (Current Setup)

**Pros:**
- No redeployment needed
- Works with current contracts

**Cons:**
- Owner must manually trigger graduation
- More gas costs
- Less automated

**How it works:**
1. Token reaches threshold
2. Owner manually calls graduationManager.createLiquidity()
3. Owner manually transfers tokens + cBTC from BondingCurve to manager

## 🚀 Recommended Path Forward

### Deploy New Contracts

```bash
# 1. Deploy new BondingCurve
forge script script/Deploy.s.sol:Deploy \
  --rpc-url https://rpc.testnet.citrea.xyz \
  --broadcast --legacy

# This will deploy:
# - New TokenFactory
# - New BondingCurve (with graduation manager support)

# 2. Set graduation manager
export PRIVATE_KEY=0xcd6e40c315aa007128416e77c85e900ad7391b8923c200ddcfd005c8ecd6e9f6
cast send <NEW_BONDING_CURVE_ADDRESS> \
  "setGraduationManager(address)" \
  0x322aEBB3e7f98F19fea672E637a6A9E619105c2A \
  --rpc-url https://rpc.testnet.citrea.xyz \
  --private-key $PRIVATE_KEY \
  --legacy

# 3. Update DEXGraduationManager to point to new BondingCurve
cast send 0x322aEBB3e7f98F19fea672E637a6A9E619105c2A \
  "setBondingCurve(address)" \
  <NEW_BONDING_CURVE_ADDRESS> \
  --rpc-url https://rpc.testnet.citrea.xyz \
  --private-key $PRIVATE_KEY \
  --legacy

# 4. Update lib/contracts.ts with new addresses

# 5. Update database event listener with new addresses

# 6. Redeploy frontend
```

## 📊 Current Status

✅ DEXGraduationManager deployed: `0x322aEBB3e7f98F19fea672E637a6A9E619105c2A`
✅ Uniswap V2 Router found: `0xFE4924Be93794fd85C0698Eb8da5014b12EBB90c`
❌ BondingCurve needs upgrade: `0x2cA9DBEB3D6931c07B0819D446Ee3276F4154cc8` (old version)
❌ TokenFactory needs update: `0x67F30a990bFa8356bFBC261971dA2AcfAF994490` (points to old curve)

## 🤔 Decision Needed

**Do you want to:**

1. **Redeploy everything** (clean, recommended for testnet)
2. **Keep old tokens, deploy new contracts for future tokens** (hybrid)
3. **Manual graduation only** (no automatic DEX listing)

Let me know and I'll proceed accordingly!
