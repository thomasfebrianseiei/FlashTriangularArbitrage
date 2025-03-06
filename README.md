# FlashTriangularArbitrage
FlashTriangularArbitrage Flow
```
sequenceDiagram
    participant User
    participant Contract as FlashTriangularArbitrage
    participant PancakeSwap
    participant BiSwap
    participant TokenA
    participant TokenB
    participant TokenC

    Note over User,Contract: Inisiasi Arbitrase
    User->>Contract: executeFlashLoan(pairAddress, borrowAmount, data, fromPancake)
    
    alt fromPancake == true
        Contract->>PancakeSwap: pair.swap(amount0Out, amount1Out, address(this), data)
        PancakeSwap-->>Contract: pancakeCall(_sender, _amount0, _amount1, _data)
        Note over Contract,PancakeSwap: Flash loan berhasil diterima
        
        Contract->>Contract: _handleFlashLoan(data)
        
        Note over Contract,TokenC: Eksekusi Arbitrase Triangular
        Contract->>PancakeSwap: swapExactTokensForTokens (TokenA->TokenB)
        PancakeSwap-->>Contract: return TokenB
        
        Contract->>BiSwap: swapExactTokensForTokens (TokenB->TokenC)
        BiSwap-->>Contract: return TokenC
        
        Contract->>PancakeSwap: swapExactTokensForTokens (TokenC->TokenA)
        PancakeSwap-->>Contract: return TokenA (jumlah lebih besar dari pinjaman)
        
        Note over Contract,PancakeSwap: Pembayaran Kembali Flash Loan
        Contract->>PancakeSwap: token.safeTransfer(pinjaman + fee)
        
    else fromPancake == false
        Contract->>BiSwap: pair.swap(amount0Out, amount1Out, address(this), data)
        BiSwap-->>Contract: BiswapCall(_sender, _amount0, _amount1, _data)
        Note over Contract,BiSwap: Flash loan berhasil diterima
        
        Contract->>Contract: _handleFlashLoan(data)
        
        Note over Contract,TokenC: Eksekusi Arbitrase Triangular
        Contract->>BiSwap: swapExactTokensForTokens (TokenA->TokenB)
        BiSwap-->>Contract: return TokenB
        
        Contract->>PancakeSwap: swapExactTokensForTokens (TokenB->TokenC)
        PancakeSwap-->>Contract: return TokenC
        
        Contract->>BiSwap: swapExactTokensForTokens (TokenC->TokenA)
        BiSwap-->>Contract: return TokenA (jumlah lebih besar dari pinjaman)
        
        Note over Contract,BiSwap: Pembayaran Kembali Flash Loan
        Contract->>BiSwap: token.safeTransfer(pinjaman + fee)
    end
    
    Note over Contract: Perhitungan & Distribusi Profit
    Contract->>Contract: Hitung platform fee
    
    alt profit > 0
        Contract->>User: token.safeTransfer(userProfit)
        Contract->>Contract: token.safeTransfer(platformFee) ke owner
        Contract->>User: emit ArbitrageExecuted event
    end
