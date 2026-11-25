# 海光算力卡兼容性检测工具

本包提供供海光 K100_AI 使用的预编译算子。运行 `compute-audit run` 可逐卡检测矩阵乘加（GEMM）、均方根归一化（RMSNorm）、缩放点积注意力（SDPA）和计算校验（Validate Compute）。终端使用英文，逐卡 Markdown 报告使用中文。

## 运行环境

- Linux x86_64 海光算力服务器，或运行在该服务器上的 GPU 容器。
- 已安装与本包匹配的海光驱动、DTK 运行库和厂商适配版 PyTorch。实测组合为 Ubuntu 22.04.5、PyTorch 2.9.0、HIP 6.3.26113 和 K100_AI（`gfx928`）。DTK 运行库和 OpenSSL 3 的 `libcrypto.so.3` 应可被动态链接器找到。
- Python 3.10 或以上版本，NumPy 1.24 或以上版本。本包使用 Python 3.10.12 和 NumPy 1.25.0 验证。下文的 `python3` 指已有 `venv` 和 `pip`，且可以导入厂商 PyTorch 和 NumPy 的解释器。
- 当前账户可访问待测 GPU 的设备节点，包括 `/dev/kfd` 和对应的 `/dev/dri/renderD*`。在容器中测试整机时，须将全部待测卡和相应设备节点提供给容器。

## 文件与安装

| 文件 | 用途 |
| --- | --- |
| `wheelhouse/compute_audit-3.1.0-py3-none-linux_x86_64.whl` | 命令行工具与预编译原生算子 |
| `profiles/conformance.json` | 默认四项检测配置，跳过 GPU FP64 检查 |
| `profiles/conformance-fp64.json` | 启用 GPU FP64 检查的四项检测配置 |
| `parameters/parameters.json` | 算子参数 |
| `SHA256SUMS` | 随包文件的 SHA-256 校验清单 |

在本仓库根目录运行。虚拟环境沿用现有 Python 环境中的厂商 PyTorch 和 NumPy，安装过程使用本地 wheel。

```sh
PACKAGE_DIR="$(pwd -P)"
sha256sum -c SHA256SUMS

VENV="$HOME/.venvs/compute-audit-3.1"
python3 -m venv --system-site-packages --without-pip "$VENV"
_PIP_USE_IMPORTLIB_METADATA=1 "$VENV/bin/python" -m pip install --no-index --no-deps --force-reinstall \
  "$PACKAGE_DIR/wheelhouse/compute_audit-3.1.0-py3-none-linux_x86_64.whl"
"$VENV/bin/compute-audit" --version
```

版本命令应显示 `compute-audit 3.1.0`。

## 检测全部物理卡

安装完成后，从本仓库根目录运行下列命令。`--devices all` 选择本机发现的全部物理卡。整机检测时，可先查看 `probe.json`，确认 `visible_devices` 与 `physical_devices` 一致。检测报告写入仓库外的新目录。

```sh
PACKAGE_DIR="$(pwd -P)"
VENV="$HOME/.venvs/compute-audit-3.1"
mkdir -p "$HOME/compute-audit-runs"
RUN_DIR="$(mktemp -d "$HOME/compute-audit-runs/run.XXXXXXXX")"

"$VENV/bin/compute-audit" probe --output "$RUN_DIR/probe.json"
"$VENV/bin/compute-audit" run --devices all \
  --profile "$PACKAGE_DIR/profiles/conformance.json" \
  --parameters "$PACKAGE_DIR/parameters/parameters.json" \
  --output "$RUN_DIR/result"
```

默认配置跳过 GPU FP64 能力探针、设备端双精度比较及六项专用自检。每卡共 36 条用例，其中 34 条必测。计时用例使用 5 次预热和 30 次计时采样。默认报告注明本轮已跳过 GPU FP64 检查，结果仅供诊断。

需要测试 GPU FP64 时，可在同一 Shell 中用下列命令单独保存结果。该配置每卡共 42 条用例，其中 40 条必测。

```sh
"$VENV/bin/compute-audit" run --devices all \
  --profile "$PACKAGE_DIR/profiles/conformance-fp64.json" \
  --parameters "$PACKAGE_DIR/parameters/parameters.json" \
  --output "$RUN_DIR/result-fp64"
```

## 查看结果

默认运行结束后，打开 `$RUN_DIR/result/report.md` 查看检测到的卡数、每张卡的四项结论和必测覆盖。机器可读结果在同目录的 `report.json`。使用 FP64 配置时，查看 `$RUN_DIR/result-fp64/report.md`。

| 终端状态 | 退出码 | 含义 |
| --- | ---: | --- |
| `PASS` | 0 | 本轮配置的必测项通过 |
| `INCOMPLETE` | 10 | 本轮覆盖或执行证据不足 |
| `FAIL` | 20 | 检测发现失败项，可在报告中定位对应的卡和算子 |
| `ERROR` 或标准错误单独显示 `Execution failed` | 30 | 执行出错，例如输出目录已经存在 |
| `UNSUPPORTED` | 40 | 当前环境或所选设备不支持本次检测 |

若某张卡的计算校验执行失败，终端还会显示该卡本轮编号及 `Validate Compute  Execution failed`。重跑时使用新的输出目录。
