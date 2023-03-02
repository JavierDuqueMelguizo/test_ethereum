

# Introducción

Vamos a desplegar una red adhoc para visualizar el comportamiento de Caliper. Para ello, utilizaremos el siguiente bloque de genesis:
```json
{
    "config": {
        "chainId": 48122,
        "homesteadBlock": 1,
        "eip150Block": 2,
        "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
        "eip155Block": 3,
        "eip158Block": 3,
        "byzantiumBlock": 4,
        "constantinopleBlock": 5,
        "clique": {
            "period": 5,
            "epoch": 30000
        }
    },
    "nonce": "0x0",
    "timestamp": "0x5ca916c6",
    "extraData": "0x0000000000000000000000000000000000000000000000000000000000000000c0A8e4D217eB85b812aeb1226fAb6F588943C2C20000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
    "gasLimit": "0x47b760",
    "difficulty": "0x1",
    "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "coinbase": "0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2",
    "number": "0x0",
    "gasUsed": "0x0",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "alloc": {
        "0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2": {
            "balance": "0x200000000000000000000000000000000000000000000000000000000000000"
        }
    }
}
```

Como detalles de esta red:
- Viene con usuario por defecto (material criptográfico sacado de [aquí](https://github.com/hyperledger/caliper-benchmarks/tree/main/networks/ethereum/1node-clique/keys)) con un balance de 158456325028.5287 ETH, que, a su vez, es el minero en el nodo (responsable de donar capacidad computo para ejecutar transacciones) de acuerdo a lo definido en `coinbase` .
**NOTA**: El material criptográfico de este usuario debe ser insertado dentro del directorio `./Caliper/ethereum/data_directory_keystore/` después de inicializar la blockchain `./binaries/geth --datadir ./Caliper/ethereum/data_directory/ --identity jdm93 init ./Caliper/ethereum/genesis/genesis.json `

- Viene con un estado de la BlockChain definido en `./Caliper/ethereum/data/bc.dat` sacado de [aquí](https://github.com/hyperledger/caliper-benchmarks/blob/main/networks/ethereum/1node-clique/data/bc.dat) . Dentro de esta blockchain viene definido un contrato inteligente, importante para hacer las pruebas con caliper más adelante. El contrato inteligente desplegado es el siguiente:
```solidity

pragma solidity >=0.4.22 <0.6.0;

contract simple {
    mapping(string => int) private accounts;

    function open(string memory acc_id, int amount) public {
        accounts[acc_id] = amount;
    }

    function query(string memory acc_id) public view returns (int amount) {
        amount = accounts[acc_id];
    }

    function transfer(string memory acc_from, string memory acc_to, int amount) public {
        accounts[acc_from] -= amount;
        accounts[acc_to] += amount;
    }
}
```
**NOTA**: Esta información fue sacada de [aquí](https://github.com/hyperledger/caliper-benchmarks/tree/main/src/ethereum/simple) . En este directorio tenemos tanto el código fuente del contrato (`simple.sol`) como el bytecode y el ABI del contrato definidos dentro de un fichero json (`simple.json`), asi como el gas necesario para el despliegue.

# Despliegue BC Privada

En el anterior apartado, definimos y/o obtuvimos todos los recursos necesarios para inicializar esta BC privada. Aquí solo vamos a recapitular los comandos que vamos a ejecutar. Para mas detalle sobre los pasos realizados consultar el documento `Despliegue_Ethereum.md`:

1. Bloque Genesis:
```bash
./binaries/geth --datadir ./Caliper/ethereum/data_directory/ --identity jdm93 init ./Caliper/ethereum/data/genesis.json 
```
2. Credenciales usuario:
```bash
echo -n "password" > ./Caliper/ethereum/data_directory/keystore/password
echo -n '{"address":"c0a8e4d217eb85b812aeb1226fab6f588943c2c2","crypto":{"cipher":"aes-128-ctr","ciphertext":"521588833e66d0e052120c30080e37e847ae7877eb09dc6760eb10382c2e6d4f","cipherparams":{"iv":"cee02dbf21538798041c99f49bef9afa"},"kdf":"scrypt","kdfparams":{"dklen":32,"n":262144,"p":1,"r":8,"salt":"916a08841725aba0bf2a2d39f489f0d5d3f2b5032067e44655b8ff542c492f03"},"mac":"f90667ea4de3e977b1cdd23ed722ddffc3f6888b21ea68cc7d5ef77463022e94"},"id":"7543fa05-dac5-4ae9-856d-e96ddea28c41","version":3}' > ./Caliper/ethereum/data_directory/keystore/UTC--2019-05-05T20-07-38.958128475Z--c0a8e4d217eb85b812aeb1226fab6f588943c2c2
```
3.  **Opcional**. Importamos estado bloque al definido en `./Caliper/ethereum/data/bc.dat` .
```bash
./binaries/geth --nousb --datadir ./Caliper/ethereum/data_directory/ import ./Caliper/ethereum/data/bc.dat
```
4. Levantamos nodo 
```bash
./binaries/geth --datadir ./Caliper/ethereum/data_directory/ --unlock 0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2 --password ./Caliper/ethereum/data_directory/keystore/password --mine --miner.threads 2 --miner.etherbase 0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2 --ws --ws.addr 0.0.0.0 --ws.origins='*' --ws.api admin,eth,miner,personal,web3 --allow-insecure-unlock --nodiscover --miner.gasprice 1
```
5. Ejecutamos ventana interactiva en otra ventana:
```bash
./binaries/geth attach --datadir ./Caliper/ethereum/data_directory/
```

# Despliegue Caliper

1. Definimos el fichero `./Caliper/ethereum/data/networkconfig.json` que define los parámetros de conexión con la red Ethereum que Caliper necesitará para la ejecución de las transacciones.Este fichero ha sido sacado de [aquí](https://github.com/hyperledger/caliper-benchmarks/blob/main/networks/ethereum/1node-clique/networkconfig.json):

```json
{
    "caliper": {
        "blockchain": "ethereum",
        "command": {
            "start": "echo 'Beginning'",
            "end": "echo 'End'"
        }
    },
    "ethereum": {
        "url": "ws://localhost:8546",
        "contractDeployerAddress": "0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2",
        "contractDeployerAddressPassword": "password",
        "fromAddress": "0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2",
        "fromAddressPassword": "password",
        "transactionConfirmationBlocks": 2,
        "contracts": {
            "simple": {
                "path": "../ethereum/data/simple.json",
                "estimateGas": true,
                "gas": {
                    "query": 100000,
                    "transfer": 70000
                }
            }
        }
    }
}
```
2. Definimos el fichero `./Caliper/ethereum/data/simple.json` que contiene los datos del contrato `simple.sol` (definido anteriormente) compilado, entre otros datos:
    <details>
    <summary>Simple.json content</summary>

    ```json
    {
        "name": "simple",
        "abi": [
            {
                "constant": false,
                "inputs": [
                    {
                        "internalType": "string",
                        "name": "acc_from",
                        "type": "string"
                    },
                    {
                        "internalType": "string",
                        "name": "acc_to",
                        "type": "string"
                    },
                    {
                        "internalType": "int256",
                        "name": "amount",
                        "type": "int256"
                    }
                ],
                "name": "transfer",
                "outputs": [],
                "payable": false,
                "stateMutability": "nonpayable",
                "type": "function"
            },
            {
                "constant": true,
                "inputs": [
                    {
                        "internalType": "string",
                        "name": "acc_id",
                        "type": "string"
                    }
                ],
                "name": "query",
                "outputs": [
                    {
                        "internalType": "int256",
                        "name": "amount",
                        "type": "int256"
                    }
                ],
                "payable": false,
                "stateMutability": "view",
                "type": "function"
            },
            {
                "constant": false,
                "inputs": [
                    {
                        "internalType": "string",
                        "name": "acc_id",
                        "type": "string"
                    },
                    {
                        "internalType": "int256",
                        "name": "amount",
                        "type": "int256"
                    }
                ],
                "name": "open",
                "outputs": [],
                "payable": false,
                "stateMutability": "nonpayable",
                "type": "function"
            }
        ],
        "bytecode": "0x608060405234801561001057600080fd5b50610542806100206000396000f3fe608060405234801561001057600080fd5b50600436106100415760003560e01c80631de45b10146100465780637c261929146101a25780639064129314610271575b600080fd5b6101a06004803603606081101561005c57600080fd5b810190808035906020019064010000000081111561007957600080fd5b82018360208201111561008b57600080fd5b803590602001918460018302840111640100000000831117156100ad57600080fd5b91908080601f016020809104026020016040519081016040528093929190818152602001838380828437600081840152601f19601f8201169050808301925050505050505091929192908035906020019064010000000081111561011057600080fd5b82018360208201111561012257600080fd5b8035906020019184600183028401116401000000008311171561014457600080fd5b91908080601f016020809104026020016040519081016040528093929190818152602001838380828437600081840152601f19601f82011690508083019250505050505050919291929080359060200190929190505050610336565b005b61025b600480360360208110156101b857600080fd5b81019080803590602001906401000000008111156101d557600080fd5b8201836020820111156101e757600080fd5b8035906020019184600183028401116401000000008311171561020957600080fd5b91908080601f016020809104026020016040519081016040528093929190818152602001838380828437600081840152601f19601f820116905080830192505050505050509192919290505050610429565b6040518082815260200191505060405180910390f35b6103346004803603604081101561028757600080fd5b81019080803590602001906401000000008111156102a457600080fd5b8201836020820111156102b657600080fd5b803590602001918460018302840111640100000000831117156102d857600080fd5b91908080601f016020809104026020016040519081016040528093929190818152602001838380828437600081840152601f19601f8201169050808301925050505050505091929192908035906020019092919050505061049b565b005b806000846040518082805190602001908083835b6020831061036d578051825260208201915060208101905060208303925061034a565b6001836020036101000a038019825116818451168082178552505050505050905001915050908152602001604051809103902060008282540392505081905550806000836040518082805190602001908083835b602083106103e457805182526020820191506020810190506020830392506103c1565b6001836020036101000a038019825116818451168082178552505050505050905001915050908152602001604051809103902060008282540192505081905550505050565b600080826040518082805190602001908083835b60208310610460578051825260208201915060208101905060208303925061043d565b6001836020036101000a0380198251168184511680821785525050505050509050019150509081526020016040518091039020549050919050565b806000836040518082805190602001908083835b602083106104d257805182526020820191506020810190506020830392506104af565b6001836020036101000a038019825116818451168082178552505050505050905001915050908152602001604051809103902081905550505056fea265627a7a72315820f89fa48c6c4d5d7df165b5e3dba04cbe258ce57a3a4ef3bab1377546b84b699f64736f6c634300050b0032",
        "gas": 4700000
    }
    ```
    </details>

3. Creamos el directorio `./Caliper/src` donde vamos a definir el proyecto que inicializará el código responsable de ejecutar benchmarking a partir del framework otorgado por **Hyperledger Caliper** e inicializamos un proyecto npm ahi:
```bash
mkdir -p ./Caliper/src; cd ./Caliper/src; npm init
```
4. Instalamos el paquete `Caliper Cli` localmente:
```bash
npm install @hyperledger/caliper-cli@0.5.0
```
5. Bindeamos el cliente a la plataforma SDK requerida (en este caso **ethereum** con la version 1.3 SDK de Caliper):
```bash
npx caliper bind --caliper-bind-sut ethereum:1.3
```
6. **Opcional** . Si queremos borrar lo realizado en el paso justo anterior, repetimos comando pero intercambiando `bind` por `unbind`:
```bash
npx caliper unbind --caliper-bind-sut ethereum:1.3
```
7. Vamos a probar los 3 métodos del contrato, por lo tanto, vamos a crear 3 tests. Esto se define en el fichero de configuración del benchmarking, `./Caliper/src/config.yaml`. Este fichero contiene el siguiente contenido:

    <details>
    <summary>config.yaml content</summary>
    ```yaml
    simpleArgs: &simple-args
    initialMoney: 10000
    moneyToTransfer: 100
    numberOfAccounts: &number-of-accounts 1010

    test:
    name: simple
    description: >-
        This is an example benchmark for Caliper, to test the backend DLT's
        performance with simple account opening & querying transactions.
    workers:
        number: 1
    rounds:
        - label: open
        description: >-
            Test description for the opening of an account through the deployed
            contract.
        txNumber: *number-of-accounts
        rateControl:
            type: fixed-rate
            opts:
            tps: 50
        workload:
            module: modules/open.js
            arguments: *simple-args
        - label: query
        description: Test description for the query performance of the deployed contract.
        txNumber: *number-of-accounts
        rateControl:
            type: fixed-rate
            opts:
            tps: 100
        workload:
            module: modules/query.js
            arguments: *simple-args
        - label: transfer
        description: Test description for transfering money between accounts.
        txNumber: 50
        rateControl:
            type: fixed-rate
            opts:
            tps: 5
        workload:
            module: modules/transfer.js
            arguments:
            <<: *simple-args
            money: 100
    ```
    </details>

Vamos a desgranar el contenido de esta configuración para entenderlo bien:
- Lo primero que vemos es lo siguiente:
```yaml
simpleArgs: &simple-args
    initialMoney: 10000
    moneyToTransfer: 100
    numberOfAccounts: &number-of-accounts 1010
```
Lo primero es la definición de un objeto con referencias `simple-args` (= simpleArgs) y `number-of-accounts` (= 1010). Es parte de las bondades del formato **.yaml** y evita definir el objeto de nuevo más adelante. De hecho, se puede ver como se utiliza más adelante, por ejemplo, en:
```yaml
...
workload:
            module: benchmarks/scenario/simple/open.js
            arguments: *simple-args # <--
...
```
- Lo siguiente que viene es la definición de los tests y es la parte que `Caliper` actualmente lee. Para mas información sobre esta parte, consultar [documentación](https://hyperledger.github.io/caliper/v0.5.0/bench-config/).
  
8. El último paso, requiere la definición de los módulos apuntados en config.yaml:
```yaml
...
        module: modules/open.js
...
        module: modules/query.js
...
        module: modules/transfer.js
...
```
Estos módulos deben tener el siguiente esqueleto implementado:
```javascript
'use strict';

const { WorkloadModuleBase } = require('@hyperledger/caliper-core');

class MyWorkload extends WorkloadModuleBase {
    constructor() {
        super();
    }

    //Called on initialization
    async initializeWorkloadModule(workerIndex, totalWorkers, roundIndex, roundArguments, sutAdapter, sutContext) {
        await super.initializeWorkloadModule(workerIndex, totalWorkers, roundIndex, roundArguments, sutAdapter, sutContext);
    }

    // Called in the moment to realice the transaction
    async submitTransaction() {
        // NOOP
    }

    // Optional. Called after the test is done to return to an initial state if needed.
    async cleanupWorkloadModule() {
        // NOOP
    }
}

function createWorkloadModule() {
    return new MyWorkload();
}

module.exports.createWorkloadModule = createWorkloadModule;

```

9. Una vez esta todo definido, podemos poner el benchmarking a ejecutar de la siguiente forma:
```bash
npx caliper launch manager \
    --caliper-bind-sut ethereum:1.3 \
    --caliper-workspace . \
    --caliper-benchconfig ./config.yaml \
    --caliper-networkconfig ../ethereum/data/networkconfig.json
```
