# DEX Graduation Implementation

## 🎯 Overview

This document describes the DEX graduation implementation for Citrea Meme Launchpad. When a token reaches the graduation threshold (0.5 cBTC market cap), it automatically creates a liquidity pool on **Satsuma DEX** (Uniswap V2 fork on Citrea).

## 📋 What Was Built

### 1. **IUniswapV2Router02 Interface** (`src/interfaces/IUniswapV2Router02.sol`)
- Standard Uniswap V2 Router interface
- Compatible with Satsuma DEX on Citrea
- Provides `addLiquidityETH()` function for creating pools

### 2. **DEXGraduationManager Contract** (`src/DEXGraduationManager.sol`)
- Separate contract to handle DEX liquidity creation
- Receives tokens + cBTC from BondingCurve upon graduation
- Creates liquidity pool on Satsuma DEX
- Burns LP tokens permanently (trustless liquidity lock)
- Emergency recovery functions for edge cases

**Key Features:**
- ✅ Non-upgradeable but replaceable (owner can change manager)
- ✅ Slippage protection (default 5%)
- ✅ Automatic LP token burning
- ✅ Returns unused funds to BondingCurve
- ✅ Emergency withdrawal for recovery

### 3. **BondingCurve Updates** (Minimal Changes)
**Added:**
- `address public graduationManager` - stores manager contract address
- `event GraduationManagerUpdated` - emits when manager changes
- `function setGraduationManager(address)` - owner can set/update manager

**Modified `_graduateToken()`:**
- Checks if `graduationManager` is set
- Transfers DEX_ALLOCATION (200M tokens) to manager
- Transfers collected cBTC to manager
- Calls `createLiquidity()` on manager
- Uses low-level call to prevent revert on failure
- Still marks token as graduated even if DEX listing fails

**Lines Changed:** ~30 lines added (out of 410 total lines)

### 4. **Deployment Script** (`script/DeployGraduationManager.s.sol`)
- Foundry deployment script
- Pre-configured with Satsuma Router address
- Outputs next steps after deployment

## 🏗️ Architecture

```
Token Reaches 0.5 cBTC Market Cap
         ↓
BondingCurve._graduateToken()
         ↓
   Mark as graduated
         ↓
Transfer 200M tokens → DEXGraduationManager
Transfer ~0.5 cBTC  → DEXGraduationManager
         ↓
DEXGraduationManager.createLiquidity()
         ↓
   Approve tokens to Satsuma Router
         ↓
Router.addLiquidityETH()
         ↓
   Creates Token/WETH Pool
         ↓
   LP Tokens → 0x...dEaD (burned)
         ↓
✅ Permanent Liquidity Pool Created on Satsuma
```

## 📍 Contract Addresses

### Citrea Testnet

| Contract | Address | Status |
|----------|---------|--------|
| **TokenFactory** | `0x67F30a990bFa8356bFBC261971dA2AcfAF994490` | ✅ Deployed |
| **BondingCurve** | `0x2cA9DBEB3D6931c07B0819D446Ee3276F4154cc8` | ✅ Deployed |
| **Satsuma Router** | `0xfE2c964EA20FAd45394f6Cdca410dAD3a9898fE2` | ✅ Verified |
| **DEXGraduationManager** | _TBD after deployment_ | ⏳ Pending |

## 🚀 Deployment Instructions

### Step 1: Deploy DEXGraduationManager

```bash
# Set environment variables
export CITREA_RPC_URL="https://rpc.testnet.citrea.xyz"
export PRIVATE_KEY="your_private_key_here"

# Deploy the contract
forge script script/DeployGraduationManager.s.sol:DeployGraduationManager \
  --rpc-url $CITREA_RPC_URL \
  --broadcast \
  --legacy

# Note: Use --legacy flag for Citrea compatibility
```

### Step 2: Set Graduation Manager in BondingCurve

```bash
# Replace <MANAGER_ADDRESS> with deployed address from Step 1
cast send 0x2cA9DBEB3D6931c07B0819D446Ee3276F4154cc8 \
  "setGraduationManager(address)" \
  <MANAGER_ADDRESS> \
  --rpc-url $CITREA_RPC_URL \
  --private-key $PRIVATE_KEY \
  --legacy
```

### Step 3: Verify Setup

```bash
# Check graduation manager is set correctly
cast call 0x2cA9DBEB3D6931c07B0819D446Ee3276F4154cc8 \
  "graduationManager()" \
  --rpc-url $CITREA_RPC_URL

# Should return the manager address from Step 1
```

### Step 4: Update Frontend Config

```typescript
// In lib/contracts.ts
export const CONTRACT_ADDRESSES = {
  // ...existing addresses...
  DEX_GRADUATION_MANAGER: '<MANAGER_ADDRESS>', // Add deployed address here
}
```

## 🧪 Testing

### Manual Testing on Testnet

1. **Create a test token:**
   ```bash
   # Use your frontend or cast command
   ```

2. **Buy tokens until graduation (0.5 cBTC total):**
   ```bash
   # Multiple buy transactions totaling 0.5 cBTC
   ```

3. **Verify graduation event:**
   ```bash
   cast logs \
     --address 0x2cA9DBEB3D6931c07B0819D446Ee3276F4154cc8 \
     --rpc-url $CITREA_RPC_URL \
     | grep TokenGraduated
   ```

4. **Check liquidity on Satsuma DEX:**
   - Visit https://www.satsuma.exchange/
   - Search for your token address
   - Verify liquidity pool exists

### Expected Behavior

**Before Graduation:**
- ✅ Token tradeable on bonding curve
- ✅ Price follows x*y=k formula
- ✅ Real cBTC reserves accumulate

**At Graduation (0.5 cBTC reached):**
- ✅ `TokenGraduated` event emitted
- ✅ 200M tokens transferred to manager
- ✅ 0.5 cBTC transferred to manager
- ✅ `LiquidityCreated` event emitted from manager
- ✅ LP tokens burned (sent to 0x...dEaD)

**After Graduation:**
- ✅ Token marked as `graduated = true`
- ✅ `buyTokens()` reverts with "Token has graduated"
- ✅ `sellTokens()` reverts with "Token has graduated"
- ✅ Token tradeable on Satsuma DEX
- ✅ Permanent liquidity pool exists

## 🔒 Security Considerations

### What's Secure

1. **Non-Reentrant:** Both BondingCurve and DEXGraduationManager use ReentrancyGuard
2. **Access Control:** Only BondingCurve can call `createLiquidity()`
3. **Slippage Protection:** 5% default slippage prevents sandwich attacks
4. **Permanent Liquidity:** LP tokens burned (can't be rug pulled)
5. **Fail-Safe:** Low-level call prevents DEX failure from blocking graduation
6. **Emergency Recovery:** Owner can recover stuck funds if needed

### Potential Risks

1. **Satsuma Router Compatibility:**
   - Risk: Satsuma might not be 100% Uniswap V2 compatible
   - Mitigation: Test thoroughly on testnet first
   - Recovery: Use emergency withdraw if it fails

2. **INIT_CODE_HASH Mismatch:**
   - Risk: Pair address calculation might fail
   - Mitigation: Contract doesn't depend on correct pair address (only for events)
   - Impact: Event shows wrong pair address, but liquidity still created

3. **Price Impact:**
   - Risk: Large graduation could move DEX prices
   - Mitigation: Slippage protection ensures acceptable ratios
   - Impact: Minimal - graduation creates initial pool

## 📊 Gas Costs (Estimates)

| Operation | Gas Cost | cBTC Cost @ 5 gwei |
|-----------|----------|-------------------|
| Deploy Manager | ~2M gas | ~0.01 cBTC |
| Set Manager in BC | ~50k gas | ~0.00025 cBTC |
| Graduation (first token) | ~500k gas | ~0.0025 cBTC |
| Graduation (subsequent) | ~300k gas | ~0.0015 cBTC |

*Note: Citrea gas prices may vary*

## 🔄 Upgrade Path

### If DEX Listing Fails

1. Deploy new DEXGraduationManager with fixes
2. Call `BondingCurve.setGraduationManager(newAddress)`
3. Old manager still holds tokens/cBTC
4. Use old manager's `emergencyWithdraw()` to recover funds
5. Future graduations use new manager

### If You Want to Change DEX

1. Deploy new manager pointing to different DEX router
2. Update BondingCurve graduation manager
3. Existing graduated tokens stay on Satsuma
4. New graduations go to new DEX

## 📝 Event Listener Updates (TODO)

The event listener needs to track:

```typescript
// Add to event listener
bondingCurve.on('TokenGraduated', async (token, finalPrice, liquidityAmount, event) => {
  console.log(`🎓 Token graduated: ${token}`)

  // Update database
  await supabase
    .from('token_metadata_cache')
    .update({ graduated: true })
    .eq('token_address', token.toLowerCase())

  // Check if liquidity was created
  const managerAddress = await bondingCurve.graduationManager()
  if (managerAddress !== '0x0000000000000000000000000000000000000000') {
    // Listen for LiquidityCreated event
    // Update database with DEX pair address
  }
})
```

## 🎨 Frontend Updates (TODO)

### Graduated Token Badge

```tsx
{token.graduated && (
  <Badge variant="success">
    🎓 Graduated - Trade on DEX
  </Badge>
)}
```

### Redirect to Satsuma

```tsx
if (token.graduated) {
  return (
    <Button
      onClick={() => window.open(`https://www.satsuma.exchange/swap?token=${token.address}`, '_blank')}
    >
      Trade on Satsuma DEX
    </Button>
  )
}
```

### Show Liquidity Pool Info

```tsx
{token.graduated && token.dexPairAddress && (
  <Link href={`https://www.satsuma.exchange/pool/${token.dexPairAddress}`}>
    View Liquidity Pool
  </Link>
)}
```

## 🐛 Troubleshooting

### Graduation Doesn't Trigger

**Check:**
1. Is graduation manager set? `cast call <BC> "graduationManager()"`
2. Is manager contract deployed at that address?
3. Does manager have correct router address?

### Liquidity Not Created

**Check:**
1. View transaction on explorer
2. Check for `LiquidityCreated` event
3. If event not emitted, low-level call failed
4. Use `emergencyWithdraw()` to recover tokens/cBTC
5. Fix and redeploy manager

### Wrong Pair Address in Event

**Not Critical:**
- Liquidity still created correctly
- Pair address calculation might use wrong INIT_CODE_HASH
- Can find correct pair on Satsuma explorer

## 📚 Resources

- [Satsuma DEX](https://www.satsuma.exchange/)
- [Citrea Explorer](https://explorer.testnet.citrea.xyz/)
- [Uniswap V2 Docs](https://docs.uniswap.org/contracts/v2/overview)
- [Citrea Documentation](https://docs.citrea.xyz/)

## ✅ Checklist Before Production

- [ ] Deploy DEXGraduationManager to testnet
- [ ] Set graduation manager in BondingCurve
- [ ] Test graduation with real token
- [ ] Verify liquidity appears on Satsuma
- [ ] Update event listener
- [ ] Update frontend UI
- [ ] Test emergency recovery
- [ ] Document deployed addresses
- [ ] Update README

## 🤝 Support

If you encounter issues:
1. Check Citrea testnet explorer for transaction details
2. Verify all addresses are correct
3. Ensure Satsuma router is still at expected address
4. Use emergency recovery if needed
5. Contact Citrea/Satsuma teams for DEX-specific issues

---

**Implementation Status:** ✅ Smart Contracts Complete | ⏳ Testing Pending | ⏳ Frontend Updates Pending
