# 修复 COD Secure Attestation 验证失败：Intel PTT 证书链错配的排查与修复

> 当 TPM 2.0 和 Secure Boot 都已启用，COD 却依然提示 `New Key Failed to be generated`，问题可能并不在 BIOS，而在 Windows 使用了与 TPM 不匹配的 Intel PTT 中间证书。

## 前言

最近在运行 Call of Duty 的 Secure Attestation 检测时，我遇到了一个比较奇怪的问题。

检测工具显示：

```text
TPM 2.0: PASS
Secure Boot: PASS
Secure Attestation: FAILED

Attestation Requisite not met
New Key Failed to be generated
BIOS Firmware Update Recommended
```

TPM 2.0 和 Secure Boot 都没有问题，BIOS 也已经更新，但安全证明始终无法通过。

起初以为是 BIOS、Intel ME 固件或者网络代理的问题。然而，经过 Windows AIK 注册测试、TPM 证书检查以及证书链对比，最终发现 Windows 使用的 Intel PTT 中间证书，与 TPM NV 存储中的证书并不一致。

**通过从自己的 TPM NV 中提取正确的证书并导入 Windows，本地证书链签名错误成功消失。**

本文记录整个排查与修复过程，包括具体命令、错误原因、验证方法及注意事项。

需要说明，本文已确认修复的是 Windows 本地 EK 证书链签名错误；COD 最终能否通过验证，还需要单独复测。

## 一、环境信息

本次故障发生在以下设备上：

| 项目 | 配置 |
|---|---|
| 操作系统 | Windows 11 Pro 25H2 |
| OS Build | 26200.9711 |
| CPU | Intel Core i7-14700KF |
| 主板 | ASUS ROG STRIX Z790-A GAMING WIFI |
| BIOS | 3202 |
| TPM | Intel PTT 2.0 |
| TPM 固件 | 600.18.1040.2765 |
| TPM Model | ADL |
| 启动模式 | UEFI |
| Secure Boot | Enabled |

需要强调，这不是仅针对华硕 Z790 的问题。其他 Intel PTT 设备也有类似报告，但 TPM 固件和 NV 索引布局可能不同。

因此，下面的操作不能跳过验证步骤直接照搬。

## 二、先确认 AIK 注册究竟失败在哪里

Windows 内置 `certreq` 工具可以尝试注册 Attestation Identity Key（AIK）证书。

以管理员身份打开 **CMD**，执行：

```cmd
certreq -enrollaik -config ""
```

这里建议使用 CMD，而不是 Windows PowerShell 5.1。PowerShell 可能无法按照预期向原生程序传递空字符串参数。

我得到的结果为：

```text
TPM-Version:2.0
VendorID:'INTC'

SubmitDone
Submit(Request): Bad Request

{
  "Message": "No valid TPM EK/Platform certificate provided in the TPM identity request message."
}

HTTP/1.1 400 Bad Request

0x80190190
HTTP_E_STATUS_BAD_REQUEST
```

这说明微软 AIK 注册服务拒绝了此次请求所提供的 TPM EK（Endorsement Key）或平台证书。

需要注意：

- 这不是简单的 TPM 未开启。
- 也不是普通的网络连接超时。
- HTTP 400 并不直接说明证书损坏。
- 需要进一步检查 Windows 本地使用的证书链。

我也尝试关闭系统代理和 TUN 后重新注册，结果仍然为相同的 HTTP 400。

因此，排查重点转向 TPM 证书链。

## 三、安装 TPM Diagnostics

微软提供了 TPM Diagnostics 可选组件，可以直接检查 EK 证书链以及 TPM NV 数据。

管理员终端执行：

```powershell
DISM /Online /Add-Capability /CapabilityName:Tpm.TpmDiagnostics~~~~0.0.1.0
```

安装完成后执行：

```powershell
TpmDiagnostics.exe ekchain
```

结果为：

```text
3 EKCerts found.
3 Intermediate Certs found.

Cert Chain
cChain: 1

Trust dwErrorStatus:0x10008

Chain 0:
cElement: 4

Trust dwErrorStatus:0x10008
```

再执行：

```powershell
TpmDiagnostics.exe ekchainNV
```

结果却是：

```text
1 EKCerts found.

Cert Chain
cChain: 1

Trust dwErrorStatus:0x10000

Chain 0:
cElement: 4

Trust dwErrorStatus:0x10000
```

这就是整个排查过程中最关键的发现。

### 两种结果代表什么？

| 状态码 | 含义 |
|---|---|
| `0x8` | CERT_TRUST_IS_NOT_SIGNATURE_VALID，证书签名无法验证 |
| `0x10000` | CERT_TRUST_IS_PARTIAL_CHAIN，无法构建至完整信任根 |
| `0x10008` | 同时包含上述两种错误 |
| `0x0` | 当前验证没有报告证书链信任错误 |

两次检查都能构建四层证书链，但是普通 `ekchain` 比 `ekchainNV` 多出了签名验证错误。

这意味着 Windows 常规证书链所选用的中间 CA，与 TPM NV 证书链可能存在差异。

不过，这还不是最终结论，需要继续比较证书本身。

## 四、对比两条 Intel PTT 证书链

Intel PTT 在我的设备上的证书结构为：

```text
TPM EK Certificate
    |
    v
CSME ADL PTT 01SVN
    |
    v
CSME ADL SVN01 Kernel CA
    |
    v
CSME ADL ROM CA
    |
    v
Intel ODCA Issuing CA
```

证书中的两个字段尤其重要：

- AKI（Authority Key Identifier）：标识签发者使用的密钥。
- SKI（Subject Key Identifier）：标识证书自身的公钥。

在通常情况下，子证书的 AKI 会与其签发 CA 的 SKI 对应。

### Windows 原先使用的中间证书

我从注册表缓存中读取了三张中间 CA 证书。

其中 PTT CA 的 SKI 为：

```text
b2a22351528fc804338ebe21db7cd89590dfcd87
```

但是 TPM EK 证书的 AKI 为：

```text
6dfced0239e4e69c8f7633ab6ce88ab949003231
```

两者并不相同。

与此同时，Windows `X509Chain` 在 EK 证书上报告：

```text
NotSignatureValid
PartialChain
```

单纯的 AKI/SKI 不匹配还不能证明签名无效，但结合实际的证书签名错误，就有必要进一步检查 TPM NV 中是否保存着另一组中间 CA。

### TPM NV 中的正确证书

后续实际提取发现，TPM NV 中 PTT CA 的 SKI 为：

```text
6dfced0239e4e69c8f7633ab6ce88ab949003231
```

恰好与 EK 证书的 AKI 对应。

进一步使用这组证书构建链后，`NotSignatureValid` 不再出现。

因此，可以确定，至少在这台设备上，Windows 原先使用的证书并不是 TPM NV 提供的那组匹配证书。

## 五、备份现有 TPM 证书配置

下面开始实际修复。

**重要：不要清除 TPM，不要重置 Secure Boot 密钥，也不要下载别人的 TPM 证书导入自己的电脑。**

后续所有中间证书都必须来自本机 TPM。

首先在管理员 PowerShell 中创建工作目录：

```powershell
$dir = "$env:USERPROFILE\Desktop\TPM-Repair"

New-Item -ItemType Directory -Force -Path $dir | Out-Null
```

备份 TPM 相关注册表缓存：

```powershell
reg export "HKLM\SYSTEM\CurrentControlSet\Services\TPM\WMI\Endorsement" "$dir\Endorsement.reg" /y
```

保存修复前的诊断结果：

```powershell
TpmDiagnostics.exe ekchain > "$dir\ekchain-before.txt"

TpmDiagnostics.exe ekchainNV > "$dir\ekchainNV-before.txt"
```

以上命令不会清除 TPM，也不会修改证书。

注册表备份并不等于备份了 TPM 内所有密钥，因此不要将它视为清除 TPM 后的完整恢复手段。

## 六、确定 TPM NV 证书存储位置

执行：

```powershell
TpmDiagnostics.exe NVSummary
```

在我的设备上，找到了：

```text
NV Public: 0x1C00100

nameAlg: TPM_ALG_SHA384
dataSize: 0x7b8 (1976)

TPMA_NV_OWNERREAD
TPMA_NV_AUTHREAD
TPMA_NV_WRITTEN
```

这表示对应 NV 索引存在，存储长度为 1976 字节。

有些设备还存在 `0x1C00101`，但我的设备并没有这个索引。

尝试读取时：

```powershell
TpmDiagnostics.exe ReadNVIndex 0x1C00101 -file "$dir\NV-101.bin"
```

返回：

```text
NVReadIndex failed with hr = 0x8028008B
```

这里无需继续尝试读取不存在的索引。

**TPM NV 索引布局并非所有机器都一致，必须以自己的 `NVSummary` 输出为准。**

## 七、从本机 TPM NV 导出证书

由于我的三张中间证书全部存储在 `0x1C00100`，直接运行：

```powershell
TpmDiagnostics.exe ReadNVIndex 0x1C00100 -file "$dir\NV-100.bin"
```

输出：

```text
NV Index data written to file.
```

检查文件大小：

```powershell
Get-Item "$dir\NV-100.bin" |
    Select-Object Name, Length
```

结果：

```text
Name         Length
----         ------
NV-100.bin     1976
```

说明 TPM NV 数据已成功读取。

需要注意，这里导出的是 TPM NV 中用于构建证书链的数据，不是 TPM 私钥。

其他设备如果证书数据分布在多个 NV 索引中，应分别读取并核对顺序，不要假定可以使用相同索引或固定大小。

## 八、从 NV 数据中提取 X.509 证书

由于 NV 数据并不一定直接是一个单独的 `.cer` 文件，需要解析其中的 DER 编码证书。

以下脚本适用于本次这种将三张 DER 证书连续存放在 NV 数据中的情况。

它会读取原始数据、识别 DER SEQUENCE、解析 X.509 证书，并导出 Subject 包含 Intel PTT 相关名称的 CA 证书。

```powershell
$dir = "$env:USERPROFILE\Desktop\TPM-Repair"
$inputFile = Join-Path $dir "NV-100.bin"
$out = Join-Path $dir "Certificates"

New-Item -ItemType Directory -Force -Path $out | Out-Null

$data = [IO.File]::ReadAllBytes($inputFile)
$found = @()

for ($i = 0; $i -lt $data.Length - 4; $i++) {
    if ($data[$i] -ne 0x30) {
        continue
    }

    $lengthByte = [int]$data[$i + 1]

    if (($lengthByte -band 0x80) -eq 0) {
        $headerLength = 2
        $contentLength = $lengthByte
    } else {
        $count = $lengthByte -band 0x7F

        if ($count -lt 1 -or $count -gt 4) {
            continue
        }

        if ($i + 2 + $count -gt $data.Length) {
            continue
        }

        $contentLength = 0

        for ($j = 0; $j -lt $count; $j++) {
            $contentLength = ($contentLength -shl 8) -bor [int]$data[$i + 2 + $j]
        }

        $headerLength = 2 + $count
    }

    $totalLength = $headerLength + $contentLength

    if ($totalLength -lt 200 -or $totalLength -gt $data.Length - $i) {
        continue
    }

    $raw = New-Object byte[] $totalLength
    [Array]::Copy($data, $i, $raw, 0, $totalLength)

    try {
        $cert = [System.Security.Cryptography.X509Certificates.X509Certificate2]::new($raw)

        if ($cert.Subject -notmatch 'CSME|Intel|ODCA') {
            continue
        }

        $basic = $cert.Extensions |
            Where-Object { $_.Oid.Value -eq '2.5.29.19' }

        if (-not ($basic -is [System.Security.Cryptography.X509Certificates.X509BasicConstraintsExtension])) {
            continue
        }

        if (-not $basic.CertificateAuthority) {
            continue
        }

        if ($found.Thumbprint -contains $cert.Thumbprint) {
            continue
        }

        $file = Join-Path $out "$($cert.Thumbprint).cer"
        [IO.File]::WriteAllBytes($file, $raw)

        $found += [pscustomobject]@{
            Subject    = $cert.Subject
            Issuer     = $cert.Issuer
            Thumbprint = $cert.Thumbprint
            File       = $file
        }

        $i += $totalLength - 1
    } catch {
        continue
    }
}

$found | Format-List
```

这段代码只进行证书识别和导出，不导入证书，也不会写入 TPM NV。

我的设备成功提取了三张证书：

| Subject | SHA-1 Thumbprint |
|---|---|
| CSME ADL PTT 01SVN | `5854C03109B6EA2BED7E4FABCD7B37686817BE6E` |
| CSME ADL SVN01 Kernel CA | `D27DD171BD1A1EB831D7E1D413DBB42D615B7ADA` |
| CSME ADL ROM CA | `863B8A2FEFC26FAE2EF6C99EEA3FF52B794AE97B` |

**这些指纹只对应本文中的设备。其他人的证书指纹可能不同，不应该拿来作为通用匹配标准。**

## 九、导入前验证证书链

不要只因为成功提取三张证书就直接导入。

首先验证它们可以被 Windows 正常解析，并且具有 CA 证书的基本约束。

```powershell
Get-ChildItem "$out\*.cer" | ForEach-Object {
    $c = [System.Security.Cryptography.X509Certificates.X509Certificate2]::new($_.FullName)

    [pscustomobject]@{
        Subject = $c.Subject
        Issuer = $c.Issuer
        Thumbprint = $c.Thumbprint
        BasicConstraints = (
            $c.Extensions |
            Where-Object { $_.Oid.Value -eq '2.5.29.19' }
        ).Format($false)
    }
} | Format-List
```

本次三张证书均显示：

```text
Subject Type=CA
Path Length Constraint=None
```

再使用 Windows 的 `X509Chain` API，临时加载这些中间证书，检查它们是否可以为当前 TPM EK 构建证书链。

```powershell
$test = [System.Security.Cryptography.X509Certificates.X509Chain]::new()

Get-ChildItem "$out\*.cer" | ForEach-Object {
    $c = [System.Security.Cryptography.X509Certificates.X509Certificate2]::new($_.FullName)

    [void]$test.ChainPolicy.ExtraStore.Add($c)
}

$test.ChainPolicy.RevocationMode = 'NoCheck'

$ek = Get-TpmEndorsementKeyInfo -HashAlgorithm sha256
$leaf = $ek.ManufacturerCertificates | Select-Object -First 1

[void]$test.Build($leaf)

$test.ChainElements | ForEach-Object {
    [pscustomobject]@{
        Subject = $_.Certificate.Subject
        Status = ($_.ChainElementStatus.Status -join ', ')
    }
} | Format-Table -AutoSize

$test.ChainStatus | Format-Table Status -AutoSize
```

我的结果：

```text
Subject                               Status
-------                               ------
TPMVersion=id:02580012...
CN=CSME ADL PTT 01SVN
CN=CSME ADL SVN01 Kernel CA
CN=CSME ADL ROM CA
```

最终：

```text
Status
------
PartialChain
```

相比原先的：

```text
NotSignatureValid
PartialChain
```

签名验证错误已经消失。

这里使用 `RevocationMode = NoCheck`，意味着没有进行证书吊销状态检查，因此这只是针对本地证书链的验证，不能当作完整的 PKI 信任认证。

`PartialChain` 仍然存在，是因为这次验证没有提供完整的 Intel ODCA 上级信任链。

但是，我们已经确认了最重要的区别：

**从 TPM NV 提取的中间证书可以构建四层 EK 证书链，并且不会出现之前的签名错误。**

## 十、导入正确的 Intel 中间证书

只有在确认导出文件来自本机 TPM NV、三张证书的签发关系对应、证书链不再报告签名错误后，才继续导入。

建议先记录当前 `Cert:\LocalMachine\CA` 的状态，以便之后判断新增了哪些证书。

然后在管理员 PowerShell 中执行：

```powershell
Get-ChildItem "$out\*.cer" | ForEach-Object {
    Import-Certificate `
        -FilePath $_.FullName `
        -CertStoreLocation Cert:\LocalMachine\CA `
        -ErrorAction Stop
}
```

这里导入的位置是：

```text
Local Computer
└── Intermediate Certification Authorities
    └── Certificates
```

也就是：

```text
Cert:\LocalMachine\CA
```

**不要将这些中间证书导入 `Cert:\LocalMachine\Root`。**

本次导入的三张证书是：

```text
CSME ADL PTT 01SVN
CSME ADL SVN01 Kernel CA
CSME ADL ROM CA
```

导入过程中，PowerShell 正常返回对应的 Subject 和 Thumbprint，未报告异常。

## 十一、验证修复结果

证书导入完成后，再次执行：

```powershell
TpmDiagnostics.exe ekchain
```

修复前：

```text
Cert Chain
cChain: 1

Trust dwErrorStatus:0x10008

Chain 0:
cElement: 4

Trust dwErrorStatus:0x10008
```

修复后：

```text
Cert Chain
cChain: 1

Trust dwErrorStatus:0x10000

Chain 0:
cElement: 4

Trust dwErrorStatus:0x10000
```

再执行：

```powershell
TpmDiagnostics.exe ekchainNV
```

结果同样为：

```text
Trust dwErrorStatus:0x10000
```

不仅错误码发生了变化，`ekchain` 现在选用的 PTT、Kernel、ROM 证书链，也与 `ekchainNV` 的输出一致。

至此可以确认：

**Windows 原先使用的不匹配 Intel PTT 中间证书已被正确证书链替代，EK 证书签名错误成功消除。**

整个过程没有清除 TPM、重置 Secure Boot 密钥、重装 Windows，也没有重新刷新 BIOS。

### 修复前后对比

| 检查项目 | 修复前 | 修复后 |
|---|---|---|
| `ekchain` | `0x10008` | `0x10000` |
| `ekchainNV` | `0x10000` | `0x10000` |
| EK 签名错误 | 存在 | 不再报告 |
| Windows 中间证书 | 与 TPM NV 不匹配 | 已使用 NV 中正确证书 |
| 完整根证书链 | 未完成 | 仍未完成 |
| COD 最终安全证明 | 失败 | 尚待复测 |

## 十二、重新进行 AIK 注册与 COD 检测

修复本地证书链以后，还需要验证微软 AIK 注册是否能够成功。

使用管理员 CMD：

```cmd
certreq -enrollaik -config ""
```

如果不再出现：

```text
No valid TPM EK/Platform certificate provided
```

说明之前的 EK 证书拒绝问题可能已经得到解决。

但如果出现新的 HTTP 错误，也必须根据新的错误内容单独分析，不能直接认为 TPM 证书链又损坏了。

随后重新运行 COD Secure Attestation Wizard，必要时按工具提示生成新的安全证明密钥。

本文记录的设备目前已经确认本地证书链错误消失，最终 AIK 注册和 COD 检测结果仍待补充。

## 十三、一些经验与注意事项

这次排查中有几个容易误判的地方。

**TPM 2.0 和 Secure Boot 正常，不意味着 AIK 注册必然成功。** TPM 基本功能、EK 证书链以及远程 Attestation 验证是不同环节。

**`certutil -verify` 的 `PartialChain` 不等于 TPM 内缺少中间证书。** Intel PTT 中间 CA 可能存在于 TPM NV 中，Windows 普通 CA 存储区未必有对应证书。

**AKI/SKI 不匹配值得调查，但不是单独的定罪证据。** 必须结合实际的数字签名验证错误和 TPM NV 对照结果。

**不能照搬其他设备的 NV 索引布局。** 本次只有 `0x1C00100`，但有的设备会将证书数据拆分存放在多个索引中。

**不要下载并导入别人的 TPM 中间证书。** 即使 Subject 名称一致，也不代表证书公钥和签名与本机 EK 对应。

**修复本地证书链不等于微软服务端必然认可它。** 远程 Attestation 可能还有额外的证书信任、注册策略及服务端验证条件。

## 总结

这次故障最初表现为：

```text
TPM 2.0: PASS
Secure Boot: PASS
Secure Attestation: FAILED
```

而真正有价值的诊断结果是：

```text
TpmDiagnostics.exe ekchain
    0x10008

TpmDiagnostics.exe ekchainNV
    0x10000
```

通过对比证书链，发现 Windows 缓存的 Intel PTT 中间证书与本机 TPM EK 并不匹配。

最终从 TPM NV 中提取正确的三张 Intel CA 证书，经验证后导入 Windows：

```text
ekchain: 0x10008 → 0x10000
```

成功消除了本地证书签名错误。

这次排查也说明，面对 TPM Attestation 故障，没有必要一开始就清除 TPM、重装系统或者反复刷新 BIOS。

**先确认错误发生在哪里，再针对性修复，往往比不断重置系统更有效。**

---

## 参考资料

- [Microsoft Learn：Import-Certificate](https://learn.microsoft.com/en-us/powershell/module/pki/import-certificate)
- [Intel Community：Call of Duty Secure Attestation Failed / Intel PTT TPM Cert](https://community.intel.com/t5/Gaming-on-Intel-Processors-with/FIX-Call-of-Duty-Secure-Attestation-Failed-New-Key-Failed-to-be/td-p/1758050)
- [Intel Community：TPM Attestation Failure](https://community.intel.com/t/tpm-attestation-failure/143370)

*本文根据 Windows 11 25H2、Intel Core i7-14700KF 和 ASUS Z790 平台的实际故障日志整理。其他设备请先核对证书链与 NV 索引，不要机械执行导入操作。*
