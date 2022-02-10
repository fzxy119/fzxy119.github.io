---
title: StarUmlV4激活
date: 2022-02-10 15:16:35
tags:uml
---

### 第一步，解包

`app.asar` 文件是 `Electron` 程序的主业务文件，是一种压缩格式的文件。我们需要修改的部分就被压缩在这里，具体文件位置为：

```
C:\Program Files\StarUML
├─locales
├─resources
| └─app.asar
└─swiftshader
```

`app.asar` 文件可以使用编辑器直接打开，但如果直接修改会导致程序无法正常运行，因此需要解包修改再压缩。

解包前需要确认您的电脑已经安装 `node.js` ，可在 `CMD` 执行以下命令，若回显版本号说明已安装，若没有安装请移步：

```
C:\Program Files>node -v
v12.18.3
```

之后全局安装 `asar` 工具

```
npm install -g asar 
或者 
cnpm install -g asar

C:\Program Files>asar -V
v3.0.3

// 出现版本号说明安装成功
```

解压 `app.asar` 文件：

```js
asar extract app.asar ./asar/
```

使用上面命令将 `app.asar` 解压到同级目录 `asar` 下，前提是 `cd` 到文件所在目录，并创建好 `asar` 文件

### 第二步，激活

解压后在 asar 目录下，找到这个文件：`asar\src\engine\license-manager.js`，使用你偏好的编辑器打开，修改其中这段代码：

```
  checkLicenseValidity () {
    this.validate().then(() => {
      setStatus(this, true)
    }, () => {
      //setStatus(this,false)  <-- comment this line
      setStatus(this, true) //<-- add this line
      //UnregisteredDialog.showDialog() <-- comment this line
    })
  }
```

注意其中注释的部分，总结来看就是将 `false` 改为 `true`，再将 `false` 的后续动作注释即可。

### 第三步，压缩

修改完成后，将修改后的内容重新打包回 `app.asar` ，使用以下命令压缩即可，其中 pack 是我前一步解压的目录：

```
asar pack asar app.asar
```

注：建议在此前备份旧的 app.asar 文件，以免造成无法挽回的损失。
