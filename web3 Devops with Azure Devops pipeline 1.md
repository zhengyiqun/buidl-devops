# Web3 Devops with Azure Devops pipeline

[toc]

---

整个技术栈涉及的工具和技术比较多，所以先拉个列表：

| 名称 | 类型 | 地址 |
| --- | --- | --- |
| Ubuntu 22.04 LTS | 操作系统 | https://releases.ubuntu.com/22.04/ |
| Docker | 开发环境 | https://docs.docker.com/engine/install/ubuntu/ |
| VSCode | 开发工具 | https://code.visualstudio.com/ |
| Goerli PoW Faucet | 以太坊测试网水龙头 | https://goerli-faucet.pk910.de/ |
| Infura | 以太坊测试网 API Gateway | https://infura.io/ |
| Solidity | 编写合约语言 | https://docs.soliditylang.org/en/v0.8.16/ |
| Truffle | 开发合约的npm toolkit | https://trufflesuite.com/ |
| Golang | 创建个人地址和发布合约 | https://goethereumbook.org/ |
| React | Dapp前端框架 | https://reactjs.org/ |
| Git | 版本管理工具 | https://git-scm.com/ |
| Azure | 微软公有云平台 | https://azure.microsoft.com/zh-cn/ |
| Azure Devops | 微软公有云开发运维平台 | https://azure.microsoft.com/en-us/services/devops/ |

### 开发环境（Docker Images）准备

Base Image 是微软打包的开发镜像，有很多个语言版本，可以直接通过[docker hub](https://hub.docker.com/_/microsoft-vscode-devcontainers)下载。我为了开发方便，基于node镜像又封装了一个镜像，加入了一些基础包。

```docker
FROM mcr.microsoft.com/vscode/devcontainers/javascript-node:latest as base

RUN apt-get update && \
    apt-get install --no-install-recommends -y \
        build-essential \
        curl && \
    rm -rf /var/lib/apt/lists/* && \
    rm -rf /etc/apt/sources.list.d/*

RUN mkdir -p /home/app
WORKDIR /home/app

RUN npm install --global web3 ethereumjs-testrpc ganache-cli truffle
```

### 开发工具（Remote Development）准备

VSCode安装完成之后，需要安装VSCode Remote插件。在插件搜索框中搜索remote，就可以看到Remote三件套:SSH、Containers、WSL。SSH和Containers就不多解释了，WSL是Windows Subsystem for Linux，如果操作系统是windows11可以直接开启WSL，通过windows docker desktop在WSL里启用docker，效果是完全一样的 

![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled.png)

*Attach到容器*

安装完插件之后，就可以看到romote图标，点击进去后切换到containers就可以看到运行中的镜像了，选中后鼠标右键Attach到镜像，就会开启一个新的vscode。这样整个开发环境就准备完成了

![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled%201.png)

### 测试项目

测试项目一共两个部分，SmartContract和Dapp。

```shell
#Smart Contract
mkdir contract
cd contract
npm init -y
truffle init

#Dapps
mkdir myapp
cd myapp
npx create-react-app myapp
```

#### Smart Contract

*测试用的账户可以提前创建，并通过 Goerli PoW Faucet 来获取测试用的ETH*

```go
package main

import (
	"context"
	"crypto/ecdsa"
	"encoding/hex"
	"fmt"
	"log"
	"math"
	"math/big"
	"os"

	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/accounts/keystore"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/common/hexutil"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/params"
	"golang.org/x/crypto/sha3"
)

func main() {

	client := InitClient()
	CreateAccountNewWallets(client)
}

func InitClient() *ethclient.Client {
        //goerli测试网API网关，可以在Infura中免费注册使用
	client, err := ethclient.Dial("https://goerli.infura.io/v3/YourApiKey")

	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("we have a connection")
	return client
}

func CreateAccountNewWallets(client *ethclient.Client) {

	privateKey, err := crypto.GenerateKey()
	if err != nil {
		log.Fatal(err)
	}

	privateKeyBytes := crypto.FromECDSA(privateKey)
        fmt.Println("privateKey")
	fmt.Println(hexutil.Encode(privateKeyBytes)[2:])

	publicKey := privateKey.Public()
	publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		log.Fatal("error casting public key to ECDSA")
	}

	publicKeyBytes := crypto.FromECDSAPub(publicKeyECDSA)
	fmt.Println("publicKeyBytes")
	fmt.Println(hexutil.Encode(publicKeyBytes)[4:])

	address := crypto.PubkeyToAddress(*publicKeyECDSA).Hex()
	fmt.Println("address")
	fmt.Println(address)

	hash := sha3.New512()
	hash.Write(publicKeyBytes[1:])
	fmt.Println(hexutil.Encode(hash.Sum(nil)[12:]))
}
```

*SmartContract 是用solidity编写的，并非标准代码，仅用于测试*

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.4.22 <0.9.0;

import "../node_modules/@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "../node_modules/@openzeppelin/contracts/access/Ownable.sol";
import "../node_modules/@openzeppelin/contracts/utils/Counters.sol";
import "../node_modules/@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "../node_modules/@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";

contract TestNFT is ERC721, ERC721Enumerable, ERC721URIStorage, Ownable{

    using Counters for Counters.Counter;
    Counters.Counter private _tokenIds;
    IERC721Enumerable public whitelistedNftContract;

    event Minted(address indexed minter, uint nftID, string uri);
  
    constructor() ERC721("TestNFT", "NFT"){}

    function mintNFT(string memory _uri, address _toAddress) public onlyOwner returns (uint256) {
 
        uint256 newItemId = _tokenIds.current();

        _mint(_toAddress, newItemId);
        _setTokenURI(newItemId, _uri);

        _tokenIds.increment();
        emit Minted(_toAddress, newItemId, _uri);
        return newItemId;
    }

    function _burn(uint256 tokenId) internal override(ERC721, ERC721URIStorage) {
        super._burn(tokenId);
    }

    function tokenURI(uint256 tokenId) public view override(ERC721, ERC721URIStorage) returns (string memory){
        return super.tokenURI(tokenId);
    }

    function _beforeTokenTransfer(address from, address to, uint256 tokenId) internal override(ERC721, ERC721Enumerable){
        super._beforeTokenTransfer(from, to, tokenId);
    }

    function supportsInterface(bytes4 interfaceId) public view override(ERC721, ERC721Enumerable) returns (bool){
        return super.supportsInterface(interfaceId);
    }
}
```

*SmartContract的测试代码*

```Javascript
const { web3 } = require("@openzeppelin/test-environment");
const { expect } = require("chai");
const { BigNumber } = require("bignumber.js");

const TestNFTContract = artifacts.require("TestNFT");

contract("TestNFT", (accounts) => {

    describe("testnft", () => {
        beforeEach(async () => {
            this.contract = await TestNFTContract.new({ from: accounts[0] });
        });
    
        it("It should mint NFT successfully", async () => {
            const tokenURI = "ipfs://QmXzG9HN7Z4kFE2yHF81Vjb2xDYu53tqhRciktrt15JpAN";
        
            const mintResult = await this.contract.mintNFT(
              tokenURI,
              accounts[0],
              { from: accounts[0] }
            );
            console.log(mintResult);
            expect(mintResult.logs[1].args.nftID.toNumber()).to.eq(0);
            expect(mintResult.logs[1].args.uri).to.eq(tokenURI);
            expect(mintResult.logs[1].args.minter).to.eq(accounts[0]);
        });
    });

    describe("owner()", () => {
        it("returns the address of the owner", async () => {
          const testntf = await TestNFTContract.deployed();
          const owner = await testntf.owner();
          assert(owner, "the current owner");
        });
    
        it("matches the address that originally deployed the contract", async () => {
          const testntf = await TestNFTContract.deployed();
          const owner = await testntf.owner();
          const expected = accounts[0];
          assert.equal(owner, expected, "matches address used to deploy contract");
        });
    });
});
```

*SmartContract编译与测试*

在封装开发镜像的时候，就已经安装了开发智能合约的工具包truffle，以下是truffle配置文件

```Javascript
require("dotenv").config();
const path = require("path");
const HDWalletProvider = require("@truffle/hdwallet-provider");
const mnemonic = process.env.MNEMONIC;

module.exports = {
  contracts_build_directory: path.join(__dirname,"build/contracts"),

  networks: {
    development: {
      //goerli测试网API网关，可以在Infura中免费注册使用
      provider: () => new HDWalletProvider(mnemonic, `https://goerli.infura.io/v3/yourapikey`),
      network_id: "5",       // Any network (default: none)
    },
  },

  // Set default mocha options here, use special reporters, etc.
  mocha: {
    reporter: 'xunit',
    reporterOptions: {
      output: 'TEST-results.xml'
    }
  },

  // Configure your compilers
  compilers: {
    solc: {
      version: "0.8.14",      // Fetch exact version from solc-bin (default: truffle's version)
    }
  },
};
```

准备好之后就可以开始合约的编译和测试了

```solidity
//编译合约
truffle compile

//测试合约
truffle test

//部署合约
truffle migrate
```

---

#### DApps(Decentralized Applications)

*Dapps是由React开发，功能很简单的前端页面，主要目的是调用之前创建的合约铸造NFT*

```Javascript
import './App.css';
import React, {Component} from "react";
import testnft from "./contracts/TestNFT.json";


class App extends Component{
  state = { storageValue: null, web3: null, account: null, toaccount: null, contract: null };

  componentDidMount = async () =>{
    try{
      //Get network provider and web3 instance.
      const Web3 = require('web3');
      const toaccount = 'to account address';
      const account = 'from account address';
      const HttpProvider ='https://goerli.infura.io/v3/yourapikey';
      const provider =  new Web3.providers.HttpProvider(HttpProvider);
      const web3 = new Web3(provider);
      //Use web3 to get the user's accounts.
      web3.eth.defaultAccount = account;
      web3.eth.accounts.wallet.add('from account private key');

      const instance = new web3.eth.Contract(
          testnft.abi,
          'address from after creating a contract',
      );

      this.setState({ web3, account, toaccount, contract: instance }, this.runExample);
    }catch(error){
      alert(
        `Failed to load web3, accounts, or contract. Check console for details.`,
      );
      console.error(error);
    }
  };

  runExample = async () => {
    const { web3, account, toaccount, contract } = this.state;
    const response = await contract.methods.mintNFT('ipfs://QmYGkmHAhySYR6zvizG3xoMEyLPs2h68swsg5BeStxL5uK', toaccount).send( {from: account, gasLimit: 3500000, gasPrice: web3.utils.toWei('1', 'Gwei')} );
    
    // Update state with the .result.
    this.setState({ storageValue: 'from: '+ response['from'] + '. to: ' + response['to'] + '. In the block: ' + response['blockNumber'] });
  };

  render() {
    if (!this.state.web3) {
      return <div>!Loading Web3, accounts, and contract...{this.state.web3},test</div>;
    }
    return (
      <div className="App">
        <h1>Good to Go!</h1>
        <p>Your Truffle Box is installed and ready.</p>
        <h2>Smart Contract Example</h2>
        <p>
          This is a simple demo of casting NFT, if successfully executed, 
          will print the <strong>from</strong> and <strong>to</strong> addresses, 
          and the <strong>blocknumber</strong>
        </p>
        <p>
          Try changing the value stored on <strong>runExample()</strong> of App.js.
        </p>
        <div>The stored value is: {this.state.storageValue}</div>
      </div>
    );
  }
}

export default App;
```

*React测试代码*

```Javascript
import { render, screen } from '@testing-library/react';
import App from './App';

test('renders learn react link', () => {
  render(<App />);
  const linkElement = screen.getByText(/Good to Go!/i);
  expect(linkElement).toBeInTheDocument();
});
```

*React编译和测试*

```Shell
#编译
npm run build

#测试
npm test --reporters=default --reporters=jest-junit
```

---

#### Azure Devops Pipeline

现在我们可以开始在Azure Devops上创建 Pipeline了。

在Azure Devops中创建新的项目，`Version control` 选择Git，

![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled%202.png)

创建好项目之后，在Repos/Files中找到`repository`的地址，点击`Generate GIt Credentials`生成Password。之后在本地设置Git连接到这个远程库

![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled%203.png)

初始化项目并推送到Remote Repository，使用上一步生成的密码，也可以使用SSH

```bash
git init
git config --global user.email "YouEmail@email.com"
git config --global user.name "YourName"
git add .
git commit -m "init project & add README file"
git remote add origin https://YourRemoteRepositoryAddressForHTTPS
git push -u origin --all
```

将代码推送到 GitHub 后，导航到 Azure DevOps  Pipelines 页面，然后单击 `Create Pipeline` 按钮

![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled%204.png)

在 `Where is your code?` 时选择`Azure Repos Git`。之后选择存放代码的repo，然后选择 `Starter pipeline`。

![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled%205.png)

Azure Pipelines 可以由[Stages、Jobs和Steps](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/stages?view=azure-devops&tabs=yaml)组成。在开始之前需要布置pipeline的Stages和Jobs。定义Stages和Jobs之间的依赖关系并查看整个pipeline。

*初始结构*

使用web editor更新代码以定义管道结构。整个pipeline有六个阶段

1、build：编译、测试和打包工件

2、dev：部署基础设施、合约和前端

3、dev_validation：等待手动验证dev并删除dev环境

4、qa：部署基础设施、合约和前端

5、qa_validation 等待手动验证 qa 并删除 qa 环境

6、prod：部署基础设施、合约和前端

在第一部分中，加入了开发环境的部署、Smart Contract和DApps的编译和测试，以及测试结果的输出。最后将Smart Contract和DApps项目文件、测试文件打包，以便后续部署使用。

```Yaml
# Starter pipeline
# Start with a minimal pipeline that you can customize to build and deploy your code.
# Add steps that build, run tests, deploy, and more:
# https://aka.ms/yaml

trigger:
- main

pool:
  vmImage: ubuntu-latest


stages:
  - stage: build
    jobs:
      - job: compile_test
        steps:
          - script: npm install --global web3 ethereumjs-testrpc ganache-cli truffle
            displayName: "install npm global package"
          - script: npm install
            displayName: "Install npm package"
            workingDirectory: $(System.DefaultWorkingDirectory)/contracts3
          - script: truffle compile
            displayName: "Compile contracts"
            workingDirectory: $(System.DefaultWorkingDirectory)/contracts3
          - script: truffle test
            displayName: "Test contracts"
            workingDirectory: $(System.DefaultWorkingDirectory)/contracts3
          - task: PublishTestResults@2
            displayName: "Publish contract test results"
            inputs:
              testRunTitle: "Contract"
              testResultsFormat: "JUnit"
              failTaskOnFailedTests: true
              testResultsFiles: "**/TEST-*.xml"
          - task: CopyFiles@2
            displayName: Package tests
            inputs: 
              Contents: |
                $(System.DefaultWorkingDirectory)/contracts3/test/**
                package.json
              TargetFolder: "$(Build.ArtifactStagingDirectory)/tests"
          - task: PublishPipelineArtifact@1
            displayName: Publish contract tests
            inputs:
              targetPath: "$(Build.ArtifactStagingDirectory)/tests"
              artifact: "tests"
              publishLocation: "pipeline"
          - task: CopyFiles@2
            displayName: Package contracts
            inputs:
              Contents: |
                $(System.DefaultWorkingDirectory)/contracts3/package.json
                $(System.DefaultWorkingDirectory)/contracts3/migrations/**
                $(System.DefaultWorkingDirectory)/contracts3/truffle-config.js
                $(System.DefaultWorkingDirectory)/contracts3/contracts/**
              TargetFolder: "$(Build.ArtifactStagingDirectory)/contracts"
          - task: PublishPipelineArtifact@1
            displayName: Publish contracts
            inputs:
              targetPath: "$(Build.ArtifactStagingDirectory)/contracts"
              artifact: "contracts"
              publishLocation: "pipeline"
          - script: npm install
            displayName: "Install frontend dependencies"
            workingDirectory: $(System.DefaultWorkingDirectory)/myapp
          - script: npm run build
            displayName: "Build frontend"
            workingDirectory: $(System.DefaultWorkingDirectory)/myapp
          - script: npm test -- --reporters=default --reporters=jest-junit
            displayName: "Test frontend"
            workingDirectory: $(System.DefaultWorkingDirectory)/myapp
            env:
              CI: true
          - task: PublishTestResults@2
            displayName: "Publish frontend test results"
            inputs:
              testRunTitle: "Frontend"
              testResultsFormat: "JUnit"
              failTaskOnFailedTests: true
              testResultsFiles: "myapp/junit*.xml"
          - task: PublishPipelineArtifact@1
            displayName: Publish frontend
            inputs:
              targetPath: "$(System.DefaultWorkingDirectory)/myapp/build"
              artifact: "client"
              publishLocation: "pipeline"
  - stage: dev
    dependsOn: build
    jobs: 
      - job: iac
      - job: deploy_contracts
        dependsOn: iac
      - job: deploy_frontend
        dependsOn: 
          - iac
          - deploy_contracts
  - stage: dev_validation
    dependsOn: dev
    jobs:
      - job: wait_for_dev_validation
      - job: delete_dev
        dependsOn: wait_for_dev_validation
  - stage: qa
    dependsOn: dev_validation
    jobs:
      - job: iac
      - job: deploy_contracts
        dependsOn: iac
      - job: deploy_frontend
        dependsOn:
          - iac
          - deploy_contracts
  - stage: qa_validation
    dependsOn: qa
    jobs:
      - job: wait_for_qa_validation
      - job: delete_qa
        dependsOn: wait_for_qa_validation
  - stage: prod
    dependsOn: qa_validation
    jobs:
      - job: iac
      - job: deploy_contracts
        dependsOn: iac
      - job: deploy_frontend
        dependsOn:
          - iac
          - deploy_contracts
```

保存并运行Pipeline，确认一切结构正确并正在运行

![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled%206.png)

项目文件打包
![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled%207.png)

DApp测试结果的输出
![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled%208.png)

测试结果的输出，所有的测试案例都通过了

![Untitled](./web3%20Devops%20with%20Azure%20Devops%20pipeline%201/Untitled%209.png)
