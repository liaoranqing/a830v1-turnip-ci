# Turnip for Adreno 830v1 on mainline DRM (Waydroid)

> 交接手册：见 [HANDOVER.md](HANDOVER.md)（系统状态、证据链、下一步、踩坑）。

在 GitHub Actions 上交叉编译适用于 **OnePlus Pad 2 Pro（Adreno 830v1，chip id `0x44050001`）**
的 Android/Bionic aarch64 Turnip 驱动，用于 Waydroid 容器（主线 DRM/msm 内核）。

## 设计

| 项目 | 内容 |
|---|---|
| Mesa 基线 | **26.2.3**（与 erhai-mesa 相同，作者补丁的天然基线） |
| 内核接口 | `-Dfreedreno-kmds=msm`（主线 DRM，**不是**手机 KGSL） |
| GPU ID | 830v1 `0xffff44050001`（补丁 + 缺失时文本注入双保险） |
| 额外修复 | erhai-mesa 的 3 个 a8xx/turnip 修复补丁 |
| 链接 | 使用 Waydroid vendor 镜像内的 bionic `libdrm.so`（ABI 与容器一致） |
| 构建 | Android NDK r29 + meson 交叉编译，产物 `libvulkan_freedreno.so` |

erhai 补丁来源：`yzddmr6/onepluspad-2pro-mainline` v0.7.1 `packaging/mesa/patches/`：

- `0001-freedreno-a830v1.patch` — 830v1 设备表项
- `turnip-gen8-tex-line-offset.patch` — gen8 TEX_LINE_OFFSET 修复
- `turnip-a8xx-runtime-unroll.patch` — A8xx 有界运行时循环展开
- `turnip-ssbo-wide-reads.patch` — 对齐只读 SSBO 的精确宽度读取

Droidloom 运行时修复来源：`denialwm/droidloom` 的 `android/aosp-patches/`（GPL-3.0-or-later，Mesa 26.1.7 基线，已在本工程 26.2.3 上实测可应用）：

- `patches/droidloom/0016-mesa-turnip-hwui-queues.patch` — Android HWUI Vulkan pipeline 需要 graphics family 两个队列（render + upload），缺失时会 abort
- `patches/droidloom/0018-mesa-ir3-alias-array-sources.patch` — ir3 `alias.tex` 保留 array/element/mask，修 DCE 崩溃

> 构建脚本对这两个补丁采用「不适用即中止构建」，避免上游漂移时静默缺件。

> 说明：上游 Mesa main 已吸收部分修复且代码已漂移，因此本工程固定在 26.2.3
> 基线，保证补丁集干净应用。

## 使用

1. 推送到你的 GitHub（见下方命令）。
2. **Actions → Build Turnip for Adreno 830v1 (msm) → Run workflow**
   - `variant`：`a830v1`（默认，最小补丁集）
   - `mesa_tarball`：默认 `https://archive.mesa3d.org/mesa-26.2.3.tar.xz`
   - `build_version`：产物版本号
3. 运行结束（约 40–70 分钟，首次下载 NDK r29 约 1GB）后，下载 artifact
   `turnip-a830v1-msm`，得到 `Turnip_A830v1-msm_V*.zip`。
   推 `v*` tag 会自动建 Release。

推送命令（在本目录执行）：

```bash
git init -b main
git add .
git commit -m "Turnip CI for Adreno 830v1 (msm backend)"
git remote add origin git@github.com:<你的用户名>/<仓库名>.git
git push -u origin main
```

## 在平板上安装与测试

```bash
unzip Turnip_A830v1-msm_V*.zip    # 得到 libvulkan_freedreno.so
sudo mkdir -p /var/lib/waydroid/overlay/vendor/lib64/hw
sudo cp libvulkan_freedreno.so /var/lib/waydroid/overlay/vendor/lib64/hw/vulkan.freedreno.so

sudo systemctl restart waydroid-container
waydroid session start

# 容器内验证（应出现 deviceGroups，且 logcat 无 "unsupported"）
sudo lxc-attach -P /var/lib/waydroid/lxc -n waydroid -- cmd gpu vkjson
```

若要让 SurfaceFlinger 走硬件渲染（UI 由 GPU 合成）：

```
# /var/lib/waydroid/waydroid_base.prop 追加/修改：
ro.hardware.egl=angle
debug.angle.backend=vulkan
```

重启容器后检查：

```bash
sudo lxc-attach -P /var/lib/waydroid/lxc -n waydroid -- \
  dumpsys SurfaceFlinger | grep -m1 GLES
# 期望：ANGLE (Adreno (TM) 830v1, Vulkan ...)
```

## 已知限制

- 只产出 **Vulkan 驱动**，不含 GLES（`libgallium_dri.so`）；UI 加速依赖 ANGLE-over-Vulkan。
- 容器 gralloc/HWC 兼容性仍需实机验证；若 SF 卡在 `RenderEngine`，说明 GPU 提交仍有问题，
  需继续叠加或移植修复。
- 若镜像的 vendor 更新导致 libdrm ABI 变化，可改动脚本里的 vendor zip URL/SHA。

## 致谢与许可

- 构建脚本与 Android/GPU 补丁集来自 [StevenMXZ/Adreno-Tools-Drivers](https://github.com/StevenMXZ/Adreno-Tools-Drivers)（GPL-3.0），本工程在其基础上改为 msm 后端并加入 830v1 相关修改。
- 830v1 与 a8xx 修复补丁来自 [yzddmr6/onepluspad-2pro-mainline](https://github.com/yzddmr6/onepluspad-2pro-mainline)。
- 本仓库沿用 GPL-3.0，见 `LICENSE`。
