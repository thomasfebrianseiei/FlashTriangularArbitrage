# FlashTriangularArbitrage
FlashTriangularArbitrage Flow
# Penjelasan Detail Kontrak FlashTriangularArbitrage

## Pendahuluan

FlashTriangularArbitrage adalah kontrak cerdas (smart contract) yang dirancang untuk melakukan arbitrase triangular antara dua bursa terdesentralisasi (DEX) - PancakeSwap dan BiSwap - di Binance Smart Chain. Kontrak ini memanfaatkan fitur flash loan untuk mendapatkan keuntungan dari perbedaan harga aset antar pasar tanpa memerlukan modal awal.

## Konsep Dasar

Arbitrase triangular adalah strategi trading yang memanfaatkan perbedaan harga antara tiga atau lebih aset untuk mendapatkan keuntungan bebas risiko. Kontrak ini mengimplementasikan strategi tersebut dengan:

1. Meminjam token dari pool likuiditas menggunakan flash loan
2. Menukar token melalui tiga jalur perdagangan berbeda untuk kembali ke token awal
3. Membayar kembali pinjaman dengan biaya
4. Mengambil selisih sebagai keuntungan

## Struktur Kontrak

### Interface & Library

```solidity
interface IERC20 {
    // Interface standar untuk token ERC20
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

library SafeERC20 {
    // Library untuk operasi token ERC20 yang aman
    function safeTransfer(IERC20 token, address to, uint256 value) internal { ... }
    function safeTransferFrom(IERC20 token, address from, address to, uint256 value) internal { ... }
    function safeApprove(IERC20 token, address spender, uint256 value) internal { ... }
}

interface IPair {
    // Interface untuk berinteraksi dengan pasangan likuiditas di DEX
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external;
    function token0() external view returns (address);
    function token1() external view returns (address);
    function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
}

interface IRouter {
    // Interface untuk berinteraksi dengan router DEX
    function swapExactTokensForTokens(...) external returns (uint[] memory amounts);
    function getAmountsOut(uint amountIn, address[] calldata path) external view returns (uint[] memory amounts);
}
```

### Variabel State Utama

```solidity
// Router DEX
IRouter public immutable pancakeRouter;
IRouter public immutable biswapRouter;

// Alamat router default pada Binance Smart Chain
address public constant PANCAKESWAP_ROUTER = 0x10ED43C718714eb63d5aA57B78B54704E256024E;
address public constant BISWAP_ROUTER = 0x3a6d8cA21D1CF76F653A67577FA0D27453350dD8;

// Alamat pemilik kontrak untuk pengambilan fee
address public owner;

// Guard untuk mencegah re-entrancy
bool private locked;

// Konfigurasi fee (basis poin - 100 = 1%)
uint256 public feePercentage = 10; // 0.1% fee
```

### Struktur Data Arbitrase

```solidity
struct ArbitrageData {
    address[] path1;        // Jalur pertukaran pertama
    address[] path2;        // Jalur pertukaran kedua
    address[] path3;        // Jalur pertukaran ketiga
    uint256[] minAmountsOut; // Jumlah minimum output yang diharapkan
    bool direction;         // true: Pancake->Biswap->Pancake, false: Biswap->Pancake->Biswap
}
```

### Event

```solidity
event ArbitrageExecuted(
    address indexed user, 
    uint256 profit, 
    uint256 loanAmount, 
    uint256 fee,
    bool direction
);

event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
event FeePercentageUpdated(uint256 oldFee, uint256 newFee);
```

## Fungsi Utama dan Alur Kerja

### 1. Konstruktor

```solidity
constructor() {
    // Inisialisasi dengan alamat router default BSC
    pancakeRouter = IRouter(PANCAKESWAP_ROUTER);
    biswapRouter = IRouter(BISWAP_ROUTER);
    owner = msg.sender;
    locked = false;
}
```

Konstruktor menginisialisasi router DEX menggunakan alamat default di Binance Smart Chain dan menetapkan pengirim transaksi sebagai pemilik.

### 2. Eksekusi Flash Loan

```solidity
function executeFlashLoan(
    address pairAddress,
    uint256 borrowAmount,
    ArbitrageData calldata data,
    bool fromPancake
) external nonReentrant { ... }
```

Fungsi ini memulai proses arbitrase:
- `pairAddress`: Alamat pasangan likuiditas untuk meminjam token
- `borrowAmount`: Jumlah token yang akan dipinjam
- `data`: Struktur data yang berisi jalur arbitrase
- `fromPancake`: Menentukan apakah meminjam dari PancakeSwap (true) atau BiSwap (false)

Setelah validasi input, fungsi memanggil `swap` pada kontrak pair untuk memulai flash loan.

### 3. Callback Flash Loan

```solidity
// Handle flash loan callback dari PancakeSwap
function pancakeCall(
    address /* _sender */,
    uint256 /* _amount0 */,
    uint256 /* _amount1 */,
    bytes calldata _data
) external nonReentrant { ... }

// Handle flash loan callback dari Biswap
function BiswapCall(
    address /* _sender */,
    uint256 /* _amount0 */,
    uint256 /* _amount1 */,
    bytes calldata _data
) external nonReentrant { ... }
```

Fungsi ini dipanggil oleh kontrak DEX setelah flash loan selesai. Setelah memvalidasi pemanggil, fungsi meneruskan data ke `_handleFlashLoan`.

### 4. Penanganan Flash Loan

```solidity
function _handleFlashLoan(bytes calldata data) internal { ... }
```

Alur fungsi ini:
1. Dekode data untuk mendapatkan detail arbitrase
2. Hitung biaya flash loan berdasarkan DEX yang digunakan
3. Jalankan arbitrase triangular
4. Bayar kembali pinjaman
5. Distribusikan keuntungan antara pengguna dan platform
6. Emit event dengan detail arbitrase

### 5. Eksekusi Arbitrase Triangular

```solidity
function _executeTriangularArbitrage(ArbitrageData memory data, uint256 loanAmount) internal returns (uint256 profit) { ... }
```

Fungsi ini melakukan tiga pertukaran token berurutan:
1. Jika direction = true:
   - Tukar token di PancakeSwap menggunakan path1
   - Tukar hasil di BiSwap menggunakan path2
   - Tukar hasil di PancakeSwap menggunakan path3
2. Jika direction = false:
   - Tukar token di BiSwap menggunakan path1
   - Tukar hasil di PancakeSwap menggunakan path2
   - Tukar hasil di BiSwap menggunakan path3

### 6. Fungsi Swap

```solidity
function _swap(
    address router, 
    address[] memory path, 
    uint256 amountIn, 
    uint256 minAmountOut
) internal returns (uint256) { ... }
```

Fungsi ini menangani pertukaran token di DEX:
1. Reset approval dan berikan approval baru untuk router
2. Panggil `swapExactTokensForTokens` pada router
3. Kembalikan jumlah token yang diterima

### 7. Perhitungan Biaya Flash Loan

```solidity
function _calculateFee(uint256 loanAmount, bool fromPancake) internal pure returns (uint256) { ... }
```

Fungsi ini menghitung biaya flash loan berdasarkan DEX yang digunakan:
- PancakeSwap: 0.3% fee - `(loanAmount * 3) / 997 + 1`
- BiSwap: 0.2% fee - `(loanAmount * 2) / 998 + 1`

### 8. Cek Profitabilitas

```solidity
function checkArbitrageProfitability(
    ArbitrageData calldata data,
    uint256 loanAmount,
    bool fromPancake
) external view returns (
    uint256 expectedProfit,
    uint256 expectedPlatformFee,
    uint256 expectedUserProfit
) { ... }
```

Fungsi ini memungkinkan pengguna mensimulasikan arbitrase sebelum menjalankannya:
1. Simulasikan semua swap untuk mendapatkan estimasi jumlah akhir
2. Hitung biaya flash loan
3. Hitung keuntungan dan fee platform
4. Kembalikan informasi profitabilitas

## Fitur Keamanan

### 1. Pencegahan Re-entrancy

```solidity
modifier nonReentrant() {
    require(!locked, "ReentrancyGuard: reentrant call");
    locked = true;
    _;
    locked = false;
}
```

Modifikator ini mencegah serangan re-entrancy dengan mengunci kontrak selama eksekusi fungsi.

### 2. SafeERC20

Kontrak menggunakan library SafeERC20 untuk operasi token yang aman:

```solidity
function safeApprove(IERC20 token, address spender, uint256 value) internal {
    // Reset approval terlebih dahulu untuk mencegah race condition
    if (value > 0 && token.allowance(address(this), spender) > 0) {
        require(token.approve(spender, 0), "SafeERC20: approve reset failed");
    }
    require(token.approve(spender, value), "SafeERC20: approve failed");
}
```

### 3. Validasi Input yang Ketat

```solidity
function executeFlashLoan(...) external nonReentrant {
    require(borrowAmount > 0, "Borrow amount must be greater than 0");
    require(data.path1.length >= 2, "Invalid path1 length");
    require(data.path2.length >= 2, "Invalid path2 length");
    require(data.path3.length >= 2, "Invalid path3 length");
    require(data.path1[0] == data.path3[data.path3.length - 1], "Path must form a loop");
    require(data.minAmountsOut.length == 3, "Invalid minAmountsOut length");
    
    // ...
}
```

### 4. Penanganan Error dengan Try-Catch

```solidity
function _getAmountOut(...) internal view returns (uint256) {
    try IRouter(router).getAmountsOut(amountIn, path) returns (uint[] memory amounts) {
        return amounts[amounts.length - 1];
    } catch {
        return 0;
    }
}
```

### 5. Verifikasi Pemanggil Callback

```solidity
function pancakeCall(...) external nonReentrant {
    require(msg.sender != address(0), "Invalid caller");
    // Verify legitimate flash loan
    IPair pair = IPair(msg.sender);
    require(pair.token0() != address(0) && pair.token1() != address(0), "Invalid pair");
    
    // ...
}
```

## Fungsi Administratif

### 1. Transfer Kepemilikan

```solidity
function transferOwnership(address newOwner) external onlyOwner {
    require(newOwner != address(0), "New owner is the zero address");
    emit OwnershipTransferred(owner, newOwner);
    owner = newOwner;
}
```

### 2. Pengaturan Fee

```solidity
function setFeePercentage(uint256 _feePercentage) external onlyOwner {
    require(_feePercentage <= 1000, "Fee too high, max 10%"); // Max fee 10%
    emit FeePercentageUpdated(feePercentage, _feePercentage);
    feePercentage = _feePercentage;
}
```

### 3. Penarikan Darurat

```solidity
function emergencyWithdraw(address token) external onlyOwner {
    uint256 balance = IERC20(token).balanceOf(address(this));
    require(balance > 0, "No tokens to withdraw");
    IERC20(token).safeTransfer(owner, balance);
}
```

## Cara Menggunakan Kontrak

### 1. Deploy Kontrak ke BSC

Kontrak sudah dikonfigurasi dengan alamat router default untuk PancakeSwap dan BiSwap di BSC.

### 2. Menyiapkan Data Arbitrase

```javascript
const arbitrageData = {
    path1: ["0x...", "0x..."],  // Min. 2 token address
    path2: ["0x...", "0x..."],  // Min. 2 token address
    path3: ["0x...", "0x..."],  // Min. 2 token address
    minAmountsOut: [amount1, amount2, amount3],  // Jumlah minimum output
    direction: true  // PancakeSwap → BiSwap → PancakeSwap
};
```

### 3. Cek Profitabilitas

```javascript
const result = await contract.checkArbitrageProfitability(
    arbitrageData,
    borrowAmount,
    fromPancake
);
```

### 4. Eksekusi Arbitrase

```javascript
await contract.executeFlashLoan(
    pairAddress,
    borrowAmount,
    arbitrageData,
    fromPancake
);
```

## Kiat dan Strategi

1. **Pemilihan Token**: 
   - Cari token dengan perbedaan harga signifikan antara PancakeSwap dan BiSwap
   - Fokus pada token dengan likuiditas tinggi untuk mengurangi slippage

2. **Jumlah Pinjaman**:
   - Mulai dengan jumlah kecil untuk pengujian
   - Sesuaikan berdasarkan kedalaman likuiditas pasar

3. **Pengaturan minAmountsOut**:
   - Gunakan fungsi `getAmountsOut` untuk mendapatkan estimasi jumlah output
   - Kurangi sedikit (0.5-2%) untuk mengakomodasi slippage

4. **Waktu Eksekusi**:
   - Cari peluang selama volatilitas pasar tinggi
   - Automatisasi pemantauan dan eksekusi untuk hasil terbaik

## Kesimpulan

Kontrak FlashTriangularArbitrage menyediakan mekanisme yang efisien untuk melakukan arbitrase triangular antara PancakeSwap dan BiSwap di Binance Smart Chain. Dengan memanfaatkan flash loan, kontrak memungkinkan pelaksanaan arbitrase tanpa modal awal yang signifikan.

Keunggulan kontrak ini meliputi:
- Dukungan untuk flash loan dari kedua DEX
- Penanganan keamanan yang komprehensif
- Perhitungan fee yang akurat untuk kedua DEX
- Fitur simulasi untuk mengecek profitabilitas sebelum eksekusi
- Alur distribusi profit yang jelas

Untuk hasil terbaik, pengguna harus memahami dinamika pasar DeFi dan mengoptimalkan parameter arbitrase berdasarkan kondisi pasar yang berlaku.
