# FlyDog: FlyDrones brain controls CRA373 walking

This repository connects the [FlyDrones](https://github.com/SpikeCalls/FlyDrones) spiking fly brain to a **trained CRA373 quadruped locomotion policy**. The fly brain produces planar velocity goals (`vx`, `vy`, `yaw rate`); the CRA373 RSL-RL policy converts those goals and robot observations into 12 joint actions. The bridge does not replace or retrain the locomotion policy.

```text
Scripted hand gesture (FlyDrones demo timeline)
  -> optic-flow illusion -> Retina -> InputEncoder
  -> MiniFly or MaleCNS LIF brain -> MotorDecoder
  -> FlyDog ground adapter (forward / lateral / yaw, escape = stop)
  -> CRA373 _commands -> trained PPO policy -> 12 joint actions -> Isaac Sim
                                           ^ robot observation and yaw feedback
```

The default demo follows FlyDrones' hand sequence: open palm, fist, move right, rush toward camera, drop hand. These inputs are **scripted**. The current CRA373 environment does not enable its front camera, so this version does not feed a robot camera image into the fly eye. The Isaac demo feeds measured yaw rate back into the FlyDrones haltere input. Adding a real robot-camera image is a later integration step.

## Repository layout

```text
FlyDog/
  src/flydog/bridge.py   Shared fly-brain controller and grounded velocity mapping
  src/flydog/demo.py     FlyDrones-style standalone preview, CSV and GIF
  isaac_demo.py          CRA373 checkpoint playback in Isaac Sim
  pyproject.toml         Standalone Python package metadata
```

FlyDog can be pushed to its own GitHub repository. It does not copy source files or model assets from the two upstream projects. Clone/install those projects alongside it. Do not upload private checkpoints unless you intend to share them.

## 1. Install the command preview

Prerequisites: Python 3.10+ and a clone of FlyDrones. From a folder containing both `FlyDog/` and `FlyDrones/`:

```powershell
cd FlyDog
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -e ..\FlyDrones
python -m pip install -e .
```

On Linux, activate with `source .venv/bin/activate` and use `../FlyDrones`.

Run the command preview without Isaac Lab or a checkpoint:

```powershell
flydog-demo --seconds 18 --live
flydog-demo --seconds 18 --record outputs/flydog.gif --csv outputs/commands.csv
```

The preview shows neural activity, velocity commands and a **kinematic command path**. The plotted marker is not a physics simulation of the dog. `--live` needs a GUI; `--record` can write a GIF without one. The MiniFly brain is FlyDrones' synthetic 850-neuron demo brain, not the full MaleCNS connectome.

## 2. 在 Isaac Lab 從零訓練 CRA373（沒有 checkpoint 時）

需要 NVIDIA Isaac Lab／Isaac Sim 與 CRA373 專案。先讓 CRA373 原本的 `scripts/rsl_rl/train.py` 能在你的 Isaac Lab 環境執行；這個訓練過程使用 CRA373 原有的 PPO 設定，**不需要 FlyDog 或 FlyDrones**。目前專案沒有附 checkpoint。

下列範例在 **PowerShell** 執行。請把 `D:\work\...` 範例路徑改成你另一台電腦上的實際位置；三個專案可以放在不同目錄。先切換到 Isaac Lab 根目錄：

```powershell
cd D:\work\IsaacLab
```

先從平地行走開始訓練。預設 `CRA373-Flat-v0` 使用 51 維觀測、12 個關節動作；`--num_envs` 可依 GPU 記憶體調小。`--max_iterations 300` 是專案目前的平地預設訓練長度，是否已能穩定行走仍須用 play 檢查。

```powershell
.\isaaclab.bat -p D:\work\CRA373\scripts\rsl_rl\train.py `
  --task CRA373-Flat-v0 --headless --num_envs 1024 --max_iterations 300
```

訓練產生的 checkpoint 位於 `D:\work\CRA373\logs\rsl_rl\cra373_flat\訓練時間資料夾\model_*.pt`。此路徑由 CRA373 的 `cra373/paths.py` 與 PPO `experiment_name` 決定；若你設了 `CRA373_LOG_ROOT`，則改用該位置。列出最近一個 checkpoint 的完整路徑：

```powershell
Get-ChildItem D:\work\CRA373\logs\rsl_rl\cra373_flat -Recurse -Filter model_*.pt |
  Sort-Object LastWriteTime -Descending | Select-Object -First 1 -ExpandProperty FullName
```

如果沒有找到檔案，先確認訓練已跑到存檔週期；目前平地 PPO 設定每 50 次 iteration 存一次。需要繼續訓練時，可使用 CRA373 `train.py` 的 `--resume --load_run <訓練時間資料夾>`。

## 3. 先單獨播放訓練好的機器狗

先確認 CRA373 策略本身可在 Isaac Sim 行走，再接上果蠅腦。這裡使用 CRA373 原有的 `play.py`，預設會尋找最新的 **RSL-RL 原始 `model_*.pt`**。

```powershell
.\isaaclab.bat -p D:\work\CRA373\scripts\rsl_rl\play.py `
  --task CRA373-Flat-v0 `
  --num_envs 1 --real-time
```

`play.py` 若省略 `--checkpoint`，會依訓練日誌搜尋最新 checkpoint；也可加上 `--checkpoint` 與上一步印出的完整路徑，指定某一版策略。不要把 `play.py` 匯出的 `exported/policy.pt` 當作 FlyDog 的 `--checkpoint`：FlyDog 載入的是 RSL-RL runner checkpoint。

## 4. 讓果蠅腦在 Isaac Sim 控制機器狗

先把 FlyDrones 和 FlyDog 安裝進 **Isaac Lab 使用的 Python 環境**。在 Isaac Lab 根目錄執行：

```powershell
.\isaaclab.bat -p -m pip install -e D:\work\FlyDrones
.\isaaclab.bat -p -m pip install -e D:\work\FlyDog
```

使用同一個 checkpoint 執行 [isaac_demo.py](isaac_demo.py)。把下面範例中的訓練時間資料夾與 `model_300.pt` 改成剛才查到的完整路徑。Isaac Sim 視窗會顯示實際物理模擬中的機器狗；終端機會列印手勢與果蠅腦產生的速度目標。

```powershell
.\isaaclab.bat -p D:\work\FlyDog\isaac_demo.py `
  --cra373-root D:\work\CRA373 `
  --checkpoint D:\work\CRA373\logs\rsl_rl\cra373_flat\2026-09-21_12-00-00\model_300.pt `
  --task CRA373-Flat-v0 --seconds 18 --real-time `
  --csv D:\work\FlyDog\outputs\isaac_trace.csv
```

Linux 範例（在 Isaac Lab 根目錄執行，替換為你自己的路徑）：

```bash
./isaaclab.sh -p /home/user/CRA373/scripts/rsl_rl/train.py --task CRA373-Flat-v0 --headless --num_envs 1024 --max_iterations 300
find /home/user/CRA373/logs/rsl_rl/cra373_flat -name 'model_*.pt' -printf '%T@ %p\n' | sort -nr | head -1 | cut -d' ' -f2-
./isaaclab.sh -p /home/user/CRA373/scripts/rsl_rl/play.py --task CRA373-Flat-v0 --num_envs 1 --real-time
./isaaclab.sh -p -m pip install -e /home/user/FlyDrones
./isaaclab.sh -p -m pip install -e /home/user/FlyDog
./isaaclab.sh -p /home/user/FlyDog/isaac_demo.py --cra373-root /home/user/CRA373 --checkpoint /home/user/CRA373/logs/rsl_rl/cra373_flat/2026-09-21_12-00-00/model_300.pt --task CRA373-Flat-v0 --seconds 18 --real-time
```

省略 `--real-time` 可讓模擬盡可能快地跑；加上 `--headless` 可不開視窗。果蠅腦預設以 20 Hz 更新，CRA373 策略則依環境步率執行。若訓練 Rough／Slope 策略，播放時也要選對應任務；Flat／Flat-Run 使用 51 維觀測，Rough／Slope 使用 238 維。

For a MaleCNS brain built by FlyDrones, replace `--brain minifly` with `--brain /path/to/malecns_brain.npz`; calibrate its readout with FlyDrones first and pass a YAML override with `decoder.readout_file` using `--fly-config`. The bridge maps FlyDrones' **throttle** readout to dog forward speed, so a readout calibrated for flight should be checked in the command preview before Isaac playback.

## Mapping and limits

| FlyDrones output | CRA373 command | Default limit |
|---|---|---:|
| `throttle + forward` | forward velocity | ±0.8 m/s |
| `lateral` | sideways velocity (sign inverted: FlyDrones right → CRA373 right) | ±0.5 m/s |
| `yaw` | yaw rate (sign inverted: FlyDrones right → CRA373 right) | ±1.0 rad/s |
| giant-fiber escape | stop all velocity commands | 0 |

The Isaac script also clamps goals to the selected CRA373 task's training command ranges. Adjust `--forward-mps` and `--yaw-rps` conservatively to stay within the trained policy's behavior. The bridge reads and writes CRA373's current `DirectRLEnv` internals (`_commands`, observation columns 12:15); changes to CRA373's observation layout require updating the bridge. The current demo is simulation-only and does not command physical hardware.
