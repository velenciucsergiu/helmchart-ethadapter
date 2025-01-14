# smartcontract

![Version: 0.1.1](https://img.shields.io/badge/Version-0.1.1-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 0.1.0](https://img.shields.io/badge/AppVersion-0.1.0-informational?style=flat-square)

A Helm chart for deploying the Smartcontract on a goquorum node and creating an OrgAccount.
OrgAccount and anchoring info of the Smartcontract will be stored in a Kubernetes Secret and ConfigMap to make them usable by other components running on Kubernetes.

## Requirements

- [helm 3](https://helm.sh/docs/intro/install/)
- URL of the Quorum node to communicate with in order to deploy the smart contract - See config parameter `config.quorumNodeUrl`

## Usage

- [Here](./README.md#values) is a full list of all configuration values.
- The [values.yaml file](./values.yaml) shows the raw view of all configuration values.

## How it works

1. A Kubernetes Job will be deployed which triggers scheduling of a Pod
2. The Pod generates an *OrgAccount* and anchors the SmartContract on the Quorum Blockchain.
3. The OrgAccount data will be stored in a Kubernetes Secret and the Smart Contract Anchoring Info (e.g. address) will being stored in a Kubernetes ConfigMap.

![How it works](./docs/smartcontract.drawio.png)

Note: Persisting these values in Kubernetes resources (Secret and ConfigMap) enables auto-configuration of *ethadapter* on a Sandbox environment.

## Installing the Chart

To install the chart with the release name `smartcontract` in namespace `ethadapter`:

```bash
helm upgrade --install smartcontract ph-ethadapter/smartcontract \
  --version=0.1.1 \
  --namespace=ethadapter --create-namespace \
  --wait --wait-for-jobs \
  --timeout 10m

```

**IMPORTANT** On a sandbox environment, install into the same namespace as *ethadapter* (usually namespace `ethadapter`). Otherwise *ethadapter* cannot auto-configure itself by reading values from Secret and ConfigMap.

## Uninstalling the Chart

To uninstall/delete the `smartcontract` deployment:

```bash
helm delete smartcontract \
  --namespace=ethadapter

```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` |  |
| config.anchoringSC | string | `"// SPDX-License-Identifier: MIT\npragma solidity >= 0.5 <= 0.7;\npragma experimental ABIEncoderV2;\n\ncontract anchoringSC {\n\n    // error codes\n    uint constant statusOK = 200;\n    uint constant statusAddedConstSSIOK = 201;\n    uint constant statusHashLinkOutOfSync = 100;\n    uint constant statusCannotUpdateReadOnlyAnchor = 101;\n    uint constant statusHashOfPublicKeyDoesntMatchControlString = 102;\n    uint constant statusSignatureCheckFailed = 103;\n\n    event InvokeStatus(uint indexed statusCode);\n\n    struct AnchorValue {\n        string newHashLinkSSI;\n        string lastHashLinkSSI;\n        string ZKPValue;\n    }\n\n    // store all anchors\n    AnchorValue[] anchorStorage;\n    //keep a mapping between anchor(keySSI) and it's versions\n    mapping (string => uint[]) anchorVersions;\n\n    //mapping between anchorID and controlString.\n    mapping (string => bytes32) anchorControlStrings;\n\n    //mapping for read only anchors\n    mapping (string => bool) readOnlyAnchorVersions;\n\n    constructor() public {\n\n    }\n\n    // public function\n    function addAnchor(string memory anchorID, bytes32 controlString,\n        string memory newHashLinkSSI, string memory ZKPValue, string memory lastHashLinkSSI,\n        bytes memory signature, bytes memory publicKey) public {\n\n        //check if thew anchorID can accept updates\n        if (isAnchorReadOnly(anchorID))\n        {\n            //anchor is readonly, reject updates\n            emit InvokeStatus(statusCannotUpdateReadOnlyAnchor);\n            return;\n        }\n\n        int validateAnchorContinuityResult = validateAnchorContinuity(anchorID, lastHashLinkSSI, newHashLinkSSI);\n        if (validateAnchorContinuityResult == 0)\n        {\n            //hash link are out of sync\n            emit InvokeStatus(statusHashLinkOutOfSync);\n            return;\n        }\n\n        if (validateAnchorContinuityResult == -1)\n        {\n            //anchor is new and we must check controlString\n            if (isEmptyBytes32(controlString))\n            {\n                //allow only one record of this anchorID\n                //this anchorID will not accept updates in the future\n                createReadOnlyNewAnchorValueOnAddAnchor(anchorID, newHashLinkSSI,ZKPValue,lastHashLinkSSI);\n                //all done, invoke ok status\n                emit InvokeStatus(statusAddedConstSSIOK);\n                return;\n            }\n            // add controlString to the mapping\n            anchorControlStrings[anchorID] = controlString;\n        }\n\n        //validate hash of the publicKey\n        if (!validatePublicKey(publicKey,anchorID))\n        {\n            emit InvokeStatus(statusHashOfPublicKeyDoesntMatchControlString);\n            return;\n        }\n\n        //validate signature\n        if (!validateSignature(anchorID, newHashLinkSSI,ZKPValue,lastHashLinkSSI, signature,publicKey))\n        {\n            emit InvokeStatus(statusSignatureCheckFailed);\n            return;\n        }\n\n        createNewAnchorValueOnAddAnchor(anchorID, newHashLinkSSI,ZKPValue,lastHashLinkSSI);\n        //all done, invoke ok status\n        emit InvokeStatus(statusOK);\n    }\n\n    function createNewAnchorValueOnAddAnchor(string memory anchorID, string memory newHashLinkSSI,\n        string memory ZKPValue, string memory lastHashLinkSSI) private\n    {\n        //create new anchor value\n        AnchorValue memory anchorValue = buildAnchorValue(newHashLinkSSI,lastHashLinkSSI,ZKPValue);\n        anchorStorage.push(anchorValue);\n        uint versionIndex = anchorStorage.length - 1;\n        //update number of versions available for that anchor\n        anchorVersions[anchorID].push(versionIndex);\n\n    }\n\n    function createReadOnlyNewAnchorValueOnAddAnchor(string memory anchorID, string memory newHashLinkSSI,\n        string memory ZKPValue, string memory lastHashLinkSSI) private\n    {\n        //for readonly anchor store the values in the same place as normal anchors\n        //getVersions will provide information the same way, regardless of the anchorID type\n\n        //create new anchor value\n        AnchorValue memory anchorValue = buildAnchorValue(newHashLinkSSI,lastHashLinkSSI,ZKPValue);\n        anchorStorage.push(anchorValue);\n        uint versionIndex = anchorStorage.length - 1;\n        //update number of versions available for that anchor\n        anchorVersions[anchorID].push(versionIndex);\n        // mark the anchorID as read only\n        readOnlyAnchorVersions[anchorID] = true;\n    }\n\n    function validatePublicKey(bytes memory publicKey,string memory anchorID) private view returns (bool){\n        return (sha256(publicKey) == anchorControlStrings[anchorID]);\n    }\n\n    function validateSignature(string memory anchorID,string memory newHashLinkSSI,string memory ZKPValue,\n        string memory lastHashLinkSSI, bytes memory signature, bytes memory publicKey) private view returns (bool)\n    {\n        return calculateAddress(publicKey) == getAddressFromHashAndSig(anchorID, newHashLinkSSI, ZKPValue, lastHashLinkSSI, signature);\n    }\n\n    // calculate the ethereum like address starting from the public key\n    function calculateAddress(bytes memory pub) private pure returns (address addr) {\n        // address is 65 bytes\n        // lose the first byte 0x04, use only the 64 bytes\n        // sha256 (64 bytes)\n        // get the 20 bytes\n        bytes memory pubk = get64(pub);\n\n        bytes32 hash = keccak256(pubk);\n        assembly {\n            mstore(0, hash)\n            addr := mload(0)\n        }\n    }\n\n    function get64(bytes memory pub) private pure returns (bytes memory)\n    {\n        //format 0x04bytes32bytes32\n        bytes32 first32;\n        bytes32 second32;\n        assembly {\n        //intentional 0x04bytes32 -> bytes32. We drop 0x04\n            first32 := mload(add(pub, 33))\n            second32 := mload(add(pub, 65))\n        }\n\n        return abi.encodePacked(first32,second32);\n    }\n\n    function getAddressFromHashAndSig(string memory anchorID,string memory newHashLinkSSI,string memory ZKPValue,\n        string memory lastHashLinkSSI, bytes memory signature) private view returns (address)\n    {\n        //return the public key derivation\n        return recover(getHashToBeChecked(anchorID,newHashLinkSSI,ZKPValue,lastHashLinkSSI), signature);\n    }\n\n    function recover(bytes32 hash, bytes memory signature) private pure returns (address)\n    {\n        bytes32 r;\n        bytes32 s;\n        uint8 v;\n\n        // Check the signature length\n        if (signature.length != 65) {\n            return (address(0));\n        }\n\n        // Divide the signature in r, s and v variables\n        // ecrecover takes the signature parameters, and the only way to get them\n        // currently is to use assembly.\n        // solium-disable-next-line security/no-inline-assembly\n        assembly {\n            r := mload(add(signature, 32))\n            s := mload(add(signature, 64))\n            v := byte(0, mload(add(signature, 96)))\n        }\n\n        // Version of signature should be 27 or 28, but 0 and 1 are also possible versions\n        if (v < 27) {\n            v += 27;\n        }\n\n        // If the version is correct return the signer address\n        if (v != 27 && v != 28) {\n            return (address(0));\n        } else {\n            // solium-disable-next-line arg-overflow\n            return ecrecover(hash, v, r, s);\n        }\n    }\n\n    function getHashToBeChecked(string memory anchorID,string memory newHashLinkSSI,string memory ZKPValue,\n        string memory lastHashLinkSSI) private view returns (bytes32)\n    {\n        //use abi.encodePacked to not pad the inputs\n        if (anchorVersions[anchorID].length == 0)\n        {\n            return sha256(abi.encodePacked(anchorID,newHashLinkSSI,ZKPValue));\n        }\n        return sha256(abi.encodePacked(anchorID,newHashLinkSSI,ZKPValue,lastHashLinkSSI));\n    }\n\n    // Used in addAnchor\n    function isAnchorReadOnly(string memory anchorID) private view returns(bool)\n    {\n        return readOnlyAnchorVersions[anchorID];\n    }\n\n    // Used in addAnchor\n    function validateAnchorContinuity(string memory anchorID, string memory lastHashLinkSSI, string memory newHashLinkSSI) private view returns (int)\n    {\n        if (anchorVersions[anchorID].length == 0)\n        {\n            //first anchor to be added\n            return -1;\n        }\n        uint index = anchorVersions[anchorID][anchorVersions[anchorID].length-1];\n\n        //can import StringUtils contract or compare hashes\n        //hash compare seems faster\n        if (sha256(bytes(anchorStorage[index].newHashLinkSSI)) == sha256(bytes(lastHashLinkSSI)))\n        {\n            //ensure we dont get double hashLinkSSI\n            if (sha256(bytes(newHashLinkSSI)) == sha256(bytes(lastHashLinkSSI)))\n            {\n                //raise out of sync. the hashlinks should be different, except 1st anchor\n                return 0;\n            }\n            //last hash link from contract is a match with the one passed\n            return 1;\n        }\n        //hash link is out of sync. the last hash link stored doesnt match with the last hash link passed\n        return 0;\n    }\n\n    function buildAnchorValue(string memory newHashLinkSSI, string memory lastHashLinkSSI, string memory ZKPValue) private pure returns (AnchorValue memory){\n        return AnchorValue(newHashLinkSSI, lastHashLinkSSI, ZKPValue);\n    }\n\n    function copyAnchorValue(AnchorValue memory anchorValue) private pure returns (AnchorValue memory){\n        return buildAnchorValue(anchorValue.newHashLinkSSI, anchorValue.lastHashLinkSSI, anchorValue.ZKPValue);\n    }\n\n    // public function\n    function getAnchorVersions(string memory anchor) public view returns (AnchorValue[] memory) {\n        if (anchorVersions[anchor].length == 0)\n        {\n            return new AnchorValue[](0);\n        }\n        uint[] memory indexList = anchorVersions[anchor];\n        AnchorValue[] memory result = new AnchorValue[] (indexList.length);\n        for (uint i=0;i<indexList.length;i++){\n            result[i] = copyAnchorValue(anchorStorage[indexList[i]]);\n        }\n\n        return result;\n\n    }\n\n    function check() public view returns (int){\n        return 0;\n    }\n\n    //utility functions\n    function isEmptyBytes32(bytes32 data) private pure returns (bool)\n    {\n        //byte32 has fixed length value\n        //some control strings are 0x0000value, so we cannot use data[0] == 0\n        return data == '';\n    }\n\n}"` | Code of the Anchoring SmartContact |
| config.configMapAnchoringInfoName | string | `"smartcontract-anchoring-info"` | Name of the ConfigMap with the anchoring info. If empty uses a generic name |
| config.quorumNodeUrl | string | `"http://quorum-member1.quorum:8545"` | URL of the Quorum node |
| config.secretOrgAccountName | string | `"smartcontract-org-account"` | Name of the Secret with the Org Account info. If empty uses a generic name |
| fullnameOverride | string | `""` |  |
| image.pullPolicy | string | `"IfNotPresent"` | Image Pull Policy of the node container |
| image.repository | string | `"node"` | The repository of the node container which creates account and deploys contract |
| image.tag | string | `"12"` | The Tag of the image of the node container |
| imagePullSecrets | list | `[]` |  |
| kubectlImage.pullPolicy | string | `"IfNotPresent"` | Image Pull Policy |
| kubectlImage.repository | string | `"bitnami/kubectl"` | The repository of the container image which creates configmap and secret |
| kubectlImage.tag | string | `"1.21.8"` | The Tag of the image containing kubectl. Minor Version should match to your Kubernetes Cluster Version. |
| nameOverride | string | `""` |  |
| nodeSelector | object | `{}` |  |
| serviceAccount.annotations | object | `{}` |  |
| serviceAccount.create | bool | `true` |  |
| serviceAccount.name | string | `""` |  |
| tolerations | list | `[]` |  |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.5.0](https://github.com/norwoodj/helm-docs/releases/v1.5.0)
