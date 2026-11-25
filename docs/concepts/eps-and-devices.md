# EP and Device

An **Execution Provider (EP)** is a pluggable backend in ONNX Runtime that claims and runs a subset of graph nodes on a specific hardware target. When ONNX Runtime loads a model it partitions the graph among the registered EPs: operators that an EP claims are dispatched to it, and the remainder fall back to the CPU EP. This design lets a single [ONNX](graphs-and-ir.md) model exploit an NPU, GPU, or CPU without any change to the graph itself.

A **device** is the hardware category that an EP targets — one of `npu`, `gpu`, or `cpu`. winml-cli exposes both levels of control: the high-level `--device` flag selects a hardware category, while the low-level `--ep` flag pins a specific ONNX Runtime provider name. In most workflows you set `--device` and let winml-cli resolve the best available EP; you reach for `--ep` when you need to compare or force a specific provider.

## EPs winml-cli supports

The table below lists every Execution Provider that winml-cli has explicit support for. EP names are the canonical ONNX Runtime strings accepted by `--ep`. You can also use the short **alias** (case-insensitive) anywhere the full name is accepted.

| EP | Alias | Device | Hardware | When to use |
|----|-------|--------|----------|-------------|
| `QNNExecutionProvider` | `qnn` | npu / gpu | Qualcomm NPU (Hexagon DSP) / Qualcomm GPU (Adreno) | Snapdragon-based Copilot+ PCs; best latency and power efficiency on Qualcomm silicon |
| `VitisAIExecutionProvider` | `vitisai` | npu | AMD NPU (XDNA) | AMD Ryzen AI platforms; targets the AMD AI Engine via the Vitis AI stack |
| `OpenVINOExecutionProvider` | `openvino` | npu / gpu / cpu | Intel CPU / GPU / NPU | Intel Core Ultra platforms; flexible device targeting across all three Intel compute types |
| `DmlExecutionProvider` | `dml` | gpu | GPU (DirectML) | Any DirectX 12 GPU on Windows; broad compatibility across AMD, Intel, and NVIDIA discrete/integrated graphics |
| `NvTensorRTRTXExecutionProvider` | `nv_tensorrt_rtx` | gpu | NVIDIA GPU (TensorRT RTX) | NVIDIA RTX GPUs; maximum throughput via TensorRT graph optimization |
| `MIGraphXExecutionProvider` | `migraphx` | gpu | AMD GPU (MIGraphX) | AMD discrete GPUs; hardware-accelerated inference via the MIGraphX graph engine |
| `CPUExecutionProvider` | `cpu` | cpu | CPU | Universal fallback; always available regardless of hardware |

To see which EPs are available on the current machine, run:

```bash
winml sys --list-ep
```

## Device vs. EP on the CLI

winml-cli exposes two overlapping flags for targeting hardware. Understanding their relationship prevents confusion when using `winml analyze`, `winml compile`, or `winml build`.

**`--device` (high-level)**

Accepts one of four values: `auto`, `cpu`, `gpu`, or `npu`. When set to `auto` (the default), winml-cli inspects the machine and selects the highest-priority device class that has a compatible EP available, in the order NPU > GPU > CPU. Setting an explicit value such as `--device npu` requests a device category without naming the EP.

For `winml analyze`, `--device` also accepts `all` — this evaluates the model against every device that has rule data, producing a side-by-side compatibility report.

```bash
# Let winml-cli pick the best available device
winml analyze --model model.onnx --device auto

# Target the NPU device class
winml analyze --model model.onnx --device npu

# Analyze against all devices at once (analyze only)
winml analyze --model model.onnx --device all
```

**`--ep` (low-level override)**

Accepts a valid EP name or alias (for example `qnn`, `vitisai`, `dml`, `openvino`), or `auto` to let winml-cli resolve the EP from the device. When `--ep` is provided with a specific value it takes precedence over `--device` and bypasses device-class resolution entirely. Use `--ep` when you need to pin a specific provider — for instance to compare `QNNExecutionProvider` against `DmlExecutionProvider` on the same machine.

For `winml analyze`, `--ep` also accepts `all` — this evaluates the model against every registered EP simultaneously.

```bash
# Force Qualcomm QNN regardless of device selection
winml analyze --model model.onnx --ep QNNExecutionProvider --device npu

# Use the short alias; winml-cli normalizes it to the full name
winml analyze --model model.onnx --ep qnn

# Analyze against all EPs at once (analyze only)
winml analyze --model model.onnx --ep all
```

The `--ep` flag accepts a free-form string and is not restricted to the choices listed above. This allows forward compatibility with EP names that winml-cli does not yet enumerate.

## Custom EP locations

Set `WINMLCLI_EP_PATH` to discover custom builds of supported EP plugins. It accepts
both directories and individual DLL files, separated by `;` on Windows:

```powershell
$env:WINMLCLI_EP_PATH = "C:\custom-ep;D:\qnn\onnxruntime_providers_qnn.dll"
winml sys --list-ep
winml perf -m "C:\models\model.onnx" --ep qnn@directory --device npu
```

Directories are scanned for known EP DLL filenames. A file entry selects only
that DLL, not other EPs in its folder. Known filenames identify the EP
case-insensitively. For arbitrary DLL filenames, specify the EP explicitly:

```powershell
$env:WINMLCLI_EP_PATH = "D:\custom\my_ep.dll;C:\other-ep"
winml perf -m "C:\models\model.onnx" --ep vitisai@directory --device npu
```

Unknown-name DLL file entries are tried for the requested EP. Known filenames
belonging to a different EP are not tried. Folder entries never scan arbitrary
DLL names. `winml sys --list-ep` ignores unknown filenames because it has no
requested EP to associate them with. Missing paths and non-DLL files produce
warnings and are skipped.

Entries are considered left to right, before default discovery. Runtime device
selection uses the first candidate that loads and exposes the requested EP and
device; a failed candidate does not prevent trying later entries.
Listing the same DLL as both a file and
through its directory does not create duplicate entries. Both forms use the
`directory` source tag, so commands supporting source pinning can use
`--ep qnn@directory`; `build` and `config` currently require plain `--ep qnn`.

Keep the plugin's dependencies available as required by its distribution. The
DLL must be compatible with the installed ONNX Runtime and process architecture;
discovery by filename does not guarantee that it can be loaded.

Explicit DLL file paths are treated literally, including brackets in filenames.
ORT GenAI validates each candidate's EP identity through regular ORT before
loading it into GenAI, since GenAI does not expose EP enumeration. Its cache
tracks both DLL path and validated EP. Failed identity checks or GenAI library
loads allow the next candidate to be tried. This preflight does not guarantee
GenAI model compatibility; that is checked when the model is loaded.

An OpenVINO build works the same way. Directories are scanned for the known
plugin filenames, for example `onnxruntime_providers_openvino_plugin.dll` for
OpenVINO and `onnxruntime_providers_qnn.dll` for QNN:

```powershell
$env:WINMLCLI_EP_PATH = "C:\path\to\openvino\build\Release"
winml perf -m "C:\path\to\model.onnx" --ep openvino@directory --device npu
```

### Plain `--ep` versus `--ep <alias>@directory`

- `--ep qnn` selects the QNN provider by precedence. Entries from
  `WINMLCLI_EP_PATH` are considered before the default sources, but other
  sources can still be used if those entries fail or do not expose the
  requested device.
- `--ep qnn@directory` restricts the request to the `directory` source, which
  is what `WINMLCLI_EP_PATH` entries produce.
- The accepted source tags are `bundled`, `pypi`, `nuget`, `msix`,
  `winml-catalog` and `directory`. An unknown tag, whitespace, or more than one
  `@` is rejected when the command line is parsed.
- `winml perf` and `winml compile` accept `--ep <alias>@<source>`. `winml build`
  and `winml config` parse it but reject it with a usage error ("does not yet
  support source pinning"); use plain `--ep` there.

### Check which DLL was selected

`winml sys --list-ep` (or `--list-ep --format json`) lists every source that
contributes each EP. In the text output each row starts with a status tag, then
the source kind (`FS` for directory entries set through `WINMLCLI_EP_PATH`), and
a `Path:` line shows the DLL that was found:

```text
Available execution providers
  QNNExecutionProvider
    [primary] FS
        Path:    C:\path\to\ep\bin\onnxruntime_providers_qnn.dll
    [shadowed] MSIX      ...
```

The layout above is taken from the renderer in `src/winml/modelkit/commands/sys.py`
and is illustrative, not captured from a run. The status tags mean:

- `primary`: registered cleanly and first in precedence for that EP.
- `shadowed`: registered cleanly but lower in precedence than a `primary` row,
  so it is not the default pick.
- `failed`: the DLL could not be registered; an `Error:` line follows.
- `incompatible`: it registered, but the machine lacks the hardware the EP
  requires (for example, no Qualcomm device for QNN).

### Troubleshooting custom EP paths

- A path that is neither a directory nor a file, or a file that is not a `.dll`,
  logs a warning and is skipped. Check the spelling and use `;` between entries.
- A directory entry only matches the known plugin filenames. If your DLL has a
  different name, list the file itself and pass `--ep <alias>@directory`.
- A DLL that cannot be registered (for example, one built for a different
  architecture than the process or an incompatible ONNX Runtime) is reported as
  `failed` in `winml sys --list-ep`, with an `Error:` line.
- If the plugin needs other DLLs, keep them where its distribution expects them;
  a load failure caused by a missing dependency is reported the same way.

Default discovery only inspects already-ready Windows ML Catalog providers; it
does not prepare or download non-ready providers. A source-pinned request such
as `--ep vitisai@directory` does not acquire a Catalog package if the requested
directory source is missing.

## See also

- [Graphs and IR](graphs-and-ir.md) — ONNX graph format, operator sets, and the IR that EPs consume
- [Weight and Activation](weight-and-activation.md) — tensor roles relevant to EP compatibility
- [winml sys](../commands/sys.md) — list available devices and EPs on the current machine
- [winml analyze](../commands/analyze.md) — check ONNX operator compatibility against a specific EP
