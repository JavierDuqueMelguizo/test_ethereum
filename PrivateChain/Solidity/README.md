
# Compilación Solidity

1. Instalar binarios:
```bash
npm install solc
```
2. Obtener ABI:
```bash
npx solc --abi smart_contracts/Storage.sol -o build
```
3. Obtener ByteCode:
```bash
npx solc --bin smart_contracts/Storage.sol -o build
```
1. Ver resultados en `./build/`.