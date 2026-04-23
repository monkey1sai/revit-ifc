# Revit IFC 繁體中文同步與發佈流程

這份文件用來整理兩件事：

1. 如何把 `Release_26.x.x` 已完成的 zh-TW 支援同步到其他版本線。
2. 如何區分「bundle 部署驗證」與「installer / package 正式發佈」兩種不同層級的交付流程。

## 適用範圍

- 這份流程以 `Release_26.x.x` 上已完成的 zh-TW 版本為基準。
- 目前已知可作為來源的 merge commit / tag：
  - merge commit：`589c2cd`
  - release tag：`IFC.v26.4.1-zhTW.1`
- 這份流程不建議直接對 `master` 做跨版本移植；應先落在對應年份的 release branch。

## 一、同步到其他版本線

### 1. 選定目標分支

- 先確認要同步的是哪一年的 Revit 版本。
- 一律從對應 release branch 開新分支，不要直接從 `master` 開始改。
- 例如要同步到 Revit 2027，應從 `origin/Release_27.x.x` 開分支，而不是從 `master` 或 2026 分支直接疊改。

範例：

```powershell
git fetch origin --prune
git switch -c feat/zh-tw-localization-r27 origin/Release_27.x.x
```

### 2. 先比對 2026 zh-TW 變更集

- 先看 2026 版本實際改了哪些檔案，不要只搬資源檔。
- 建議直接用 `fork/Release_26.x.x` 或 tag `IFC.v26.4.1-zhTW.1` 當來源。

範例：

```powershell
git diff --stat origin/Release_26.x.x...fork/Release_26.x.x
git log --oneline origin/Release_26.x.x..fork/Release_26.x.x
```

### 3. 搬移語系與專案檔設定

至少要確認下列檔案是否一併移植：

- `Source/IFCExporterUIOverride/Properties/Resources.zh-TW.resx`
- `Source/Revit.IFC.Export/Properties/Resources.zh-TW.resx`
- `Source/Revit.IFC.Import/Properties/Resources.zh-TW.resx`
- `Source/IFCExporterUIOverride/IFCExporterUIOverride.csproj`
- `Source/Revit.IFC.Export/Revit.IFC.Export.csproj`
- `Source/Revit.IFC.Import/Revit.IFC.Import.csproj`

如果目標版本的建置環境也有相同問題，通常還需要一起檢查：

- `global.json`
- `VSProps/Revit.Common.Sdk.props`
- `Source/IFCExporterUIOverride/IFCExporterUI.props`
- `Source/Revit.IFC.Export/Utility/ExporterUtil.cs`

### 4. 更新版本相關常數

同步到新版本時，不要只改資源檔，還要檢查所有年份與版本號是否一致：

- `VSProps/Revit.Common.Sdk.props`
  - `RevitAPIDir`
  - 本機安裝版 fallback 路徑
  - intermediate output root
- `Install\Program Files to Install\bundle\PackageContents.xml`
  - `AppVersion`
  - `SeriesMin`
  - `SeriesMax`
  - `Contents\<year>` 路徑
- `Install\RevitIFCSetupWix\Product.wxs`
  - `Product Name`
  - `Version`
  - 目標年份目錄，例如 `2027`
- `Install\RevitIFCSetupWix\buildInstaller.bat`
  - MSI 輸出檔名

### 5. 建置目標版本

先完成最小可驗證建置，再做 UI 驗證。

範例：

```powershell
dotnet msbuild Revit.IFC.sln /t:Build /p:Configuration=Release /p:Platform=x64
```

如果 repo 對 SDK / Revit API 路徑有版本依賴，先確認：

- `global.json` 指到可用 SDK
- `RevitAPIDir` 指到正確年份的 Revit API

### 6. 部署 bundle 並驗證

完成建置後，先做 bundle 層級驗證，而不是直接假設 installer 已經完成。

至少要確認：

- `C:\ProgramData\Autodesk\ApplicationPlugins\IFC <year>.bundle\Contents\<year>` 有新的主 DLL
- `zh-TW` 目錄存在
- 下列 satellite resource DLL 都已部署：
  - `IFCExporterUIOverride.resources.dll`
  - `Revit.IFC.Export.resources.dll`
  - `Revit.IFC.Import.resources.dll`
- `Revit.IFC.addin` 仍正確指向 bundle 內的 DLL

### 7. 做實際 UI 驗證

bundle 檔案存在不代表語系一定載入成功，還要進 Revit 做人工檢查。

建議至少驗證：

- IFC Export UI 是否出現繁體中文
- IFC Import / Link 相關字串是否出現繁體中文
- 沒有因為缺字串而退回英文或顯示空白鍵值

### 8. PR 一律打到對應 release branch

- 2026 改動打到 `Release_26.x.x`
- 2027 改動打到 `Release_27.x.x`
- 不要把單一年份的在地化修正直接開到 `master`

### 9. merge 後再補 tag / release

- tag 應該打在對應 release branch 的 merge commit 上
- 不要把 2026 的 tag 直接掛到 `master`
- release note 要標明：
  - 基於哪個官方版本
  - 新增哪些 zh-TW 資源
  - 是否只完成 bundle 驗證，還是已完成 installer 發佈驗證

## 二、bundle 部署流程

這是目前最直接、最可驗證的交付層級。

### 1. 建置輸出

先完成 `Release|x64` 建置。

### 2. 複製主 DLL

至少要部署：

- `Revit.IFC.Common.dll`
- `Revit.IFC.Export.dll`
- `Revit.IFC.Import.Core.dll`
- `Revit.IFC.Import.dll`
- `IFCExporterUIOverride.dll`

### 3. 複製 zh-TW resource DLL

至少要部署：

- `zh-TW\IFCExporterUIOverride.resources.dll`
- `zh-TW\Revit.IFC.Export.resources.dll`
- `zh-TW\Revit.IFC.Import.resources.dll`

### 4. 驗證 addin manifest

檢查 `Revit.IFC.addin` 仍然指向 bundle 內的 DLL，而不是舊版安裝目錄或其他年份目錄。

### 5. 啟動 Revit 做人工驗證

只有做到這一步，才能宣告 zh-TW 外掛在該年份版本真正可用。

## 三、installer / package 發佈流程

這一段和 bundle 部署不同。bundle 驗證通過，不代表 MSI / package 已經完整在地化。

### 目前 repo 現況

截至目前為止，`master` 上的 installer / package 設定有兩個重要限制：

- `Install\Program Files to Install\bundle\PackageContents.xml` 目前仍是 `SupportedLocales="Enu"`
- `Install\RevitIFCSetupWix\Product.wxs` 目前只明確包入：
  - `ProductFRFiles`
  - `ProductDEFiles`

也就是說，目前 WiX 專案沒有現成的 `zh-TW` installer 項目可以直接沿用。

### 如果只是內部部署或驗證

- 可以只走 bundle 部署流程
- 不必先把 MSI 包版也做完

### 如果要做正式 MSI / package 發佈

需要補以下項目：

1. 更新年份與版本號
   - `PackageContents.xml`
   - `Product.wxs`
   - `buildInstaller.bat`

2. 新增 zh-TW 安裝目錄
   - 參照 `INSTALLFRUI` / `INSTALLDEUI`
   - 新增對應的 `INSTALLZHTWUI`

3. 新增 zh-TW component group
   - 參照 `ProductFRFiles` / `ProductDEFiles`
   - 新增例如 `ProductZhTWFiles`

4. 把 zh-TW satellite resource DLL 納入 MSI
   - `IFCExporterUIOverride.resources.dll`
   - `Revit.IFC.Export.resources.dll`
   - `Revit.IFC.Import.resources.dll`

5. 把新 component group 掛到 product feature
   - 在 `ProductFeature` 下加入 `ComponentGroupRef`

6. 重新檢查 `SupportedLocales`
   - 這個值不要直接猜
   - 應依 Autodesk 既有 package 規範或實際發佈要求確認後再改

7. 建 MSI 並做乾淨環境驗證
   - 安裝到乾淨機器或乾淨 VM
   - 確認安裝後 bundle 結構、resource DLL、UI 語系都正確

## 四、建議交付順序

如果之後要把 zh-TW 功能擴到另一個年份，建議順序如下：

1. 從對應 release branch 開功能分支
2. 移植 zh-TW 資源與必要建置修正
3. 完成 solution build
4. 完成 bundle 部署
5. 完成實機 UI 驗證
6. 開 PR 到對應 release branch
7. merge 後補 tag / release note
8. 若該版本需要正式安裝包，再補 WiX / package 設定

## 五、避免踩雷

- 不要把 2026 的 release tag 掛到 `master`
- 不要只複製 `IFCExporterUIOverride.resources.dll` 就當成 zh-TW 已完成
- 不要把 bundle 驗證和 MSI 發佈混成同一件事
- 不要在未確認 `SupportedLocales` 規則前，直接硬改 locale code
- 不要跳過 Revit 內的人工 UI 驗證
