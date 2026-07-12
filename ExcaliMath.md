/*
ExcaliMath: a Math Sidepanel to use LaTeX as if it was normal text

1. Advanced LaTeX Editor (uses latex-suite if available)
2. Scientific Graphing Editor
3. Saved Templates Library for quick reuse
4. Live editing on the canvas
5. Automatic detection of selected LaTeX images or Math Graph lines on the canvas to edit them

/!\ This script requires Excalidraw Extras Mathjax

```js 
*/

if (!ea.verifyMinimumPluginVersion || !ea.verifyMinimumPluginVersion("2.25.3")) {
    new Notice("This script requires a newer version of Excalidraw. Please install the latest version.");
    return;
}

// ---------------------------------------------------------
// 1. Localization & Strings
// ---------------------------------------------------------
const LOCALE = (localStorage.getItem("language") || "en").toLowerCase();

const STRINGS = {
  en: {
    TAB_FORMULA: "Formula",
    TAB_GRAPH: "Graph",
    TAB_LIBRARY: "Library",
    FORMULA_INPUT_PLACEHOLDER: "Enter LaTeX formula here...\nExample: \\sum_{i=1}^n i = \\frac{n(n+1)}{2}",
    INSERT: "Insert",
    UPDATE: "Update Selected",
    SCALE: "Scale",
    GRAPH_TYPE: "Function Type",
    GRAPH_FORMULA: "f(x) =",
    GRAPH_POLY: "Coefficients (highest power first)",
    GRAPH_GAUSS: "Mean (μ) / StdDev (σ)",
    GRAPH_POISSON: "Lambda (λ)",
    DOMAIN: "Domain (X)",
    CODOMAIN: "CoDomain (Y)",
    SCALE_XY: "Pixel Scale (X / Y)",
    RESOLUTION: "Resolution",
    SHOW_AXES: "Show Axes",
    AXES_COLOR: "Axes Color",
    CLOSE_PLOT: "Close Plot (Connect ends)",
    STROKE_COLOR: "Color",
    FONT : "Font",
    STROKE_WIDTH: "Stroke Width",
    ROUGHNESS: "Roughness",
    ADD_TO_LIBRARY: "Save to Library",
    ERROR_MATHJAX: "Excalidraw-Extras (MathJax) is required to render LaTeX formulas. Please install and enable it.",
    INFO_LATEX_SUITE: "Install 'Obsidian LaTeX Suite' for faster formula entry.",
    LIBRARY_EMPTY: "Library is empty. Save formulas or graphs to see them here.",
    LOAD: "Load",
    INVALID_FORMULA: "Invalid formula. Cannot render preview.",
    PROMPT_NAME: "Name for this preset:",
    CUSTOM_FORMULA: "Custom f(x)",
    POLYNOMIAL: "Polynomial",
    GAUSSIAN: "Gaussian (Normal)",
    POISSON: "Poisson",
    PREVIEW_TITLE: "Live Preview",
    DELETE: "Delete"
  },
  zh: {
    TAB_FORMULA: "公式",
    TAB_GRAPH: "图表",
    TAB_LIBRARY: "库",
    FORMULA_INPUT_PLACEHOLDER: "在此输入LaTeX公式...\n例如: \\sum_{i=1}^n i = \\frac{n(n+1)}{2}",
    INSERT: "插入",
    UPDATE: "更新已选",
    SCALE: "缩放",
    GRAPH_TYPE: "函数类型",
    GRAPH_FORMULA: "f(x) =",
    GRAPH_POLY: "系数 (最高次幂在前)",
    GRAPH_GAUSS: "均值 (μ) / 标准差 (σ)",
    GRAPH_POISSON: "Lambda (λ)",
    DOMAIN: "定义域 (X)",
    CODOMAIN: "到达域",
    SCALE_XY: "像素缩放 (X / Y)",
    RESOLUTION: "分辨率",
    SHOW_AXES: "显示坐标轴",
    AXES_COLOR: "坐标轴颜色",
    CLOSE_PLOT: "闭合图表 (连接首尾)",
    STROKE_COLOR: "线条颜色",
    FONT : "字体",
    STROKE_WIDTH: "线条粗细",
    ROUGHNESS: "粗糙度",
    ADD_TO_LIBRARY: "保存至库",
    ERROR_MATHJAX: "需要安装并启用 Excalidraw-Extras (MathJax) 插件才能渲染 LaTeX 公式。",
    INFO_LATEX_SUITE: "建议安装 'Obsidian LaTeX Suite' 以加快公式输入。",
    LIBRARY_EMPTY: "库为空。保存公式或图表后将在此显示。",
    LOAD: "加载",
    INVALID_FORMULA: "公式无效。无法渲染预览。",
    PROMPT_NAME: "为该预设命名：",
    CUSTOM_FORMULA: "自定义 f(x)",
    POLYNOMIAL: "多项式",
    GAUSSIAN: "高斯分布 (正态)",
    POISSON: "泊松分布",
    PREVIEW_TITLE: "实时预览",
    DELETE: "删除"
  }
};
const t = (key) => STRINGS[LOCALE]?.[key] ?? STRINGS.en[key] ?? key;

const SCALE_PRESETS = [
  ["XS", 0.75],
  ["S", 1],
  ["M", 1.35],
  ["L", 1.7],
  ["XL", 2.4],
];

const COLOR_PRESETS = [
  "#1e1e1e",
  "#e03131",
  "#2f9e44",
  "#1971c2",
  "#f08c00",
];

const FONT_PRESETS = [
    { label: "Normal", wrapper: "", html: ea.obsidian.getIcon("pencil").outerHTML, },
    { label: "Sans", wrapper: "sf", html: '<span style="font-family:sans-serif">A</span>'},
    { label: "Code", wrapper: "tt", html: '<span style="font-size:10px">&lt;/&gt;</span>' },
    { label: "Bold", wrapper: "bf", html: '<span style="font-weight:900;font-size:17px;">B</span>', },
    { label: "Italic", wrapper: "it", html: "<i>I</i>" },
    { label: "Cal", wrapper: "cal", html: "𝒜" },
];

// ---------------------------------------------------------
// 2. Global State Management
// ---------------------------------------------------------
const GRAPH_DEFAULTS = {
  custom: { customFormula: "Math.sin(x) * x", xMin: -10, xMax: 10, xScale: 30, yScale: 30, resolution: 100, showAxes: true, axesColor: "#888888", closePlot: false, strokeColor: "#1e1e1e", strokeWidth: 2, roughness: 0 },
  polynomial: { polyCoeffs: "1, 0, -5", xMin: -4, xMax: 4, xScale: 50, yScale: 10, resolution: 100, showAxes: true, axesColor: "#888888", closePlot: false, strokeColor: "#1e1e1e", strokeWidth: 2, roughness: 0 },
  gaussian: { gaussMean: 0, gaussStdDev: 1, xMin: -4, xMax: 4, xScale: 80, yScale: 300, resolution: 100, showAxes: true, axesColor: "#888888", closePlot: false, strokeColor: "#1e1e1e", strokeWidth: 2, roughness: 0 },
  poisson: { poissonLambda: 4, xMin: 0, xMax: 12, xScale: 30, yScale: 300, resolution: 100, showAxes: true, axesColor: "#888888", closePlot: false, strokeColor: "#1e1e1e", strokeWidth: 2, roughness: 0 }
};

const DEFAULT_LATEX = "PlaceHolder";

let state = {
  activeTab: "formula",
  editTargetId: [], // ids from latex selected elements
  selectedId : [], // ids from all selected elements
  formula: {
    text: "\\sum_{i=1}^n i = \\frac{n(n+1)}{2}",
    color: "#1e1e1e",
    font : "",
    scale: 3
  },
  graphParams: JSON.parse(JSON.stringify(GRAPH_DEFAULTS)),
  library: []
};

let globalContentEl = null;
let previewTimeout = null;
let myEditorView = null;
let shouldFocusEditor = true;
const cmContainer = document.body.createDiv({ cls: "excalimath-cm-container excalidraw-LatexPrompt" });

const hasMathJax = !!app.plugins.plugins["excalidraw-extras"]?.api.features.isActive("mathjax");
const hasLatexSuite = !!app.plugins.plugins["obsidian-latex-suite"];

async function createEditorView() {
  const CM = ea.getCM6();
  if (CM) {
    const extensions = [
      CM.keymap.of([
        { key: "Shift-Enter",
          run: () => {
            canvaTakeFocus();
            return true;
          },
        },
      ]),
      ...ea.getMathEditorExtensions(),
      CM.EditorView.lineWrapping,
      CM.EditorView.theme({
      "&": { height: "100%" },
      ".cm-scroller": { overflow: "auto" }
      }),
    ];
    // Add update listener to sync state dynamically and trigger preview renders
    extensions.push(CM.EditorView.updateListener.of((update) => {
      if (update.docChanged) {
        const newVal = update.state.doc.toString();
        if (newVal !== state.formula.text) {
          state.formula.text = newVal;
          scheduleRender();
        }
      }
    }));
    myEditorView = new CM.EditorView({
        state: CM.EditorState.create({
            doc: state.formula.text,
            extensions: extensions
        }),
        parent: cmContainer
    });
    myEditorView.dom.addEventListener("focusout", () => {
        const newVal = myEditorView.state.doc.toString();
        if (newVal !== state.formula.text) {
            state.formula.text = newVal;
            clearTimeout(renderTimeout);
            updateFormulae();
            saveSettings();
        }
    });

    // Auto-focus the editor when the tab renders
    // setTimeout(() => myEditorView.focus(), 100);
  } else {
    // Fallback to TextArea if CM extraction fails
    const textAreaSetting = new ea.obsidian.Setting(cmContainer).setClass("excalimath-setting");
    textAreaSetting.addTextArea(text => {
      text.setValue(state.formula.text);
      text.inputEl.style.width = "100%";
      text.inputEl.style.height = "120px";
      text.inputEl.style.fontFamily = "monospace";
      text.setPlaceholder(t("FORMULA_INPUT_PLACEHOLDER"));
      
      text.onChange(val => {
        state.formula.text = val;
        updatePreviewArea();
        saveSettings();
      });

      // Auto-focus the textarea when the tab renders
      // setTimeout(() => text.inputEl.focus(), 100);
    });
    textAreaSetting.settingEl.style.display = "block";
    textAreaSetting.controlEl.style.width = "100%";
  }
}

async function loadSettings() {
  const settings = ea.getScriptSettings();
  if (settings["ExcaliMath"]) {
    try {
      const saved = JSON.parse(settings["ExcaliMath"].value);
      // Migrate old graph settings if graphParams doesn't exist
      if (!saved.graphParams && saved.graph) {
         saved.graphParams = JSON.parse(JSON.stringify(GRAPH_DEFAULTS));
         saved.graphParams[saved.graph.type || "custom"] = { ...saved.graphParams[saved.graph.type || "custom"], ...saved.graph };
      }
      state = {
        ...state,
        ...saved,
        formula: {
            ...state.formula,
            ...saved.formula,
        },
        graph : {
          ... state.graph,
          ... saved.graph,
        }
      };
      state.editTargetId = []; 
    } catch(e) {}
  }
}

async function saveSettings() {
  const settings = ea.getScriptSettings();
  settings["ExcaliMath"] = { value: JSON.stringify(state) };
  await ea.setScriptSettings(settings);
}

//----------------------------
// methods to manipulate the latex text
// adding/removing colors/font
//----------------------------

// add "\color{color}" at the begining of str if it does not contain a color yet
function addColor(str, color) {
  return "\\color{" + color + "}" + str
}

function addFont(str, font) {
  if (font !== ""){
    return "\\" + font + " " + str 
  }else{
    return str
  }
}

function getColor(str) {
	const regex = /^\\color{([#A-Za-z0-9]*)}/g;
  let matches = regex.exec(str);
  if (matches)
	  return matches[1];
  else
    return "#1e1e1e";
}

function getFont(str) {
  const regex = /^\\(sf|tt|bf|tt|it|cal)/g;
  let matches = regex.exec(str);
  if (matches)
	  return matches[1];
  else
    return "";
}

// remove the begining "\color{color}" of str if there is one
function removeColor(str){
	const regex = /^\\color{[#A-Za-z0-9]*}/g;
	return str.replace(regex, "");
}
// remove the begining "\font" of str if there is one
function removeFont(str){
	const regex = /^\\(sf|tt|bf|tt|it|cal) /g;
	return str.replace(regex, "");
}


async function getScale(el, latex){
  const dataurl = await ea.tex2dataURL(latex);
  if (dataurl && dataurl.size.height > 0 && dataurl.size.width > 0) {
    return (el.width / dataurl.size.width).toFixed(2);
  }
  return 1
}


async function getFinalLatex(text) {
  if (state.formula.color && state.formula.font !== undefined) {
    const tmp = removeFont(removeColor(text));
    return addColor(addFont(tmp, state.formula.font), state.formula.color);
  }
  return text;
}

// ---------------------------------------------------------
// 3. UI Construction & Rendering
// ---------------------------------------------------------
function injectCSS(contentEl) {
  contentEl.createEl("style", { text: `
    .excalimath-sidepanel { display: flex; flex-direction: column; height: 100%; padding: 10px; overflow-y: auto; overflow-x: hidden; }
    .excalimath-header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 10px; border-bottom: 1px solid var(--background-modifier-border); padding-bottom: 10px; }
    .excalimath-header h2 { margin: 0; display: flex; align-items: center; gap: 8px;}
    .excalimath-warning { padding: 8px; border-radius: 4px; display: flex; align-items: center; gap: 8px; margin-bottom: 10px; font-size: 0.9em; background-color: var(--background-modifier-error-hover); color: var(--text-error); border: 1px solid var(--background-modifier-error); }
    .excalimath-info { background: var(--background-secondary-alt); color: var(--text-muted); padding: 8px; border-radius: 4px; display: flex; align-items: center; gap: 8px; margin-bottom: 10px; font-size: 0.9em; }
    .excalimath-tabs { display: flex; gap: 4px; border-bottom: 1px solid var(--background-modifier-border); margin-bottom: 10px; flex-shrink: 0; }
    .excalimath-tabs button { flex: 1; border-radius: 4px 4px 0 0; border: 1px solid transparent; border-bottom: none; background: transparent; padding: 8px; box-shadow: none; cursor: pointer; color: var(--text-muted); }
    .excalimath-tabs button:hover { background: var(--background-modifier-hover); color: var(--text-normal); }
    .excalimath-tabs button.is-active { background: var(--background-secondary); border-color: var(--background-modifier-border); color: var(--text-normal); font-weight: bold; }
    .excalimath-preview-wrapper { flex: 0 0 180px; min-height: 180px; border: 1px solid var(--background-modifier-border); border-radius: 8px; display: flex; align-items: center; justify-content: center; margin: 15px 0; overflow: hidden; padding: 10px; position: relative; }
    .excalimath-preview-title { position: absolute; top: 4px; left: 8px; font-size: 0.75em; color: var(--text-muted); text-transform: uppercase; letter-spacing: 1px; z-index: 10;}
    .excalimath-preview-container { width: 100%; height: 100%; display: flex; align-items: center; justify-content: center; }
    .excalimath-preview-container svg, .excalimath-preview-container img { width: 100%; height: 100%; object-fit: contain; }
    .excalimath-tab-content { flex: 1; padding-right: 5px; display: flex; flex-direction: column; gap: 10px; overflow: visible; }
    .excalimath-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: auto; padding-top: 15px; border-top: 1px solid var(--background-modifier-border); flex-shrink: 0; }
    .excalimath-lib-card { background: var(--background-secondary); border: 1px solid var(--background-modifier-border); border-radius: 6px; padding: 12px; margin-bottom: 10px; }
    .excalimath-lib-card strong { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; color: var(--text-accent); }
    .excalimath-lib-preview { width: 100%; height: 80px; background: var(--background-primary); border-radius: 4px; margin-bottom: 10px; display: flex; align-items: center; justify-content: center; padding: 4px; border: 1px solid var(--background-modifier-border);}
    .excalimath-lib-preview img { max-width: 100%; max-height: 100%; object-fit: contain; }
    .excalimath-lib-actions { display: flex; justify-content: space-between; gap: 8px; }
    .excalimath-lib-actions button { display: flex; align-items: center; justify-content: center; gap: 4px; }
    .excalimath-grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; width: 100%; }
    .excalimath-setting { border: none !important; padding: 0 !important; }
    .excalimath-setting .setting-item-info { flex: 1; }
    .excalimath-setting .setting-item-control { flex: 1; justify-content: flex-end; }
    .excalimath-cm-container { width: 100%; overflow: visible !important; }
    .excalidraw-sidepanel-tab__content.excalimath-sidepanel .excalimath-cm-container.excalidraw-LatexPrompt .cm-tooltip-cursor {display: none !important;}

    .excalimath-selection-indicator {
      width: 10px;
      height: 10px;
      border-radius: 50%;
      background: var(--text-faint);
      border: 1px solid var(--background-modifier-border);
      display: inline-block;
      margin-left: 4px;
      transition: background-color 120ms ease;
    }

    .excalimath-selection-indicator.is-selected {
      background: var(--color-green);
    }
    .excalimath-scale-presets {
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    gap: 4px;
    margin-top: -15px;
    margin-bottom: 8px;
    }

    .excalimath-scale-btn {
        padding: 2px 0;
        font-size: 11px;
        min-height: 24px;
    }
    .excalimath-scale-btn.is-active {
        background: var(--interactive-accent);
        color: var(--text-on-accent);
        border-color: var(--interactive-accent);
    }
    .excalimath-color-presets {
    display: grid;
    grid-template-columns: repeat(6, 1fr);
    gap: 4px;
    margin-top: -6px;
    margin-bottom: 8px;
    }

    .excalimath-color-btn,
    .excalimath-color-picker {
        width: 100%;
        height: 24px;
        border: 1px solid var(--background-modifier-border);
        border-radius: 4px;
        cursor: pointer;
        padding: 0;
    }

    .excalimath-color-picker {
        background: none;
        overflow: hidden;
    }

    .excalimath-color-picker::-webkit-color-swatch-wrapper {
        padding: 0;
    }

    .excalimath-color-picker::-webkit-color-swatch {
        border: none;
    }

    .excalimath-color-btn.is-active {
        outline: 2px solid var(--interactive-accent);
        outline-offset: 1px;
    }
    .excalimath-font-presets {
      display: flex;
      margin-top: -13px;
      margin-bottom: 8px;
    }

    .excalimath-font-btn {
        flex: 1;

        height: 30px;
        padding: 0;

        border: none;
        border-right: 1px solid var(--background-modifier-border);

        background: var(--background-secondary);
        color: var(--text-normal);

        font-size: 16px;
        transition: background 120ms;
    }

    .excalimath-font-btn:first-child {
        border-radius: 8px 0 0 8px;
    }

    .excalimath-font-btn:last-child {
        border-radius: 0 8px 8px 0;
        border-right: none;
    }

    .excalimath-font-btn:hover:not(.is-active) {
        background: var(--background-modifier-hover);
    }

    .excalimath-font-btn.is-active {
        background: var(--interactive-accent);
        color: var(--text-on-accent);
    }
    .excalimath-new-latex-btn {
        display: block;
        width: 100%;
        margin-bottom: 12px;
        padding: 8px 0;
        border: none;
        border-radius: 6px;
        font-weight: bold;
        cursor: pointer;
        transition: background 120ms;
    }

    .excalimath-new-latex-btn:hover {
        background: #94d49b;
    }
        .excalimath-advanced {
        margin: 8px 0 4px 0;
        border: 1px solid var(--background-modifier-border);
        border-radius: 6px;
        overflow: hidden;
    }

    .excalimath-advanced-header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 6px 10px;
        cursor: pointer;
        user-select: none;
        background: var(--background-secondary);
        font-size: 13px;
        font-weight: 600;
    }

    .excalimath-advanced-header:hover {
        background: var(--background-modifier-hover);
    }

    .excalimath-advanced-chevron {
        font-size: 10px;
        color: var(--text-muted);
        transition: transform 120ms;
    }

    .excalimath-advanced-body {
        padding: 4px 10px 2px 10px;
    }
  `});
}

function updateSelectionIndicator() {
  const indicator = globalContentEl?.querySelector(".excalimath-selection-indicator");
  if (!indicator) return;

  const selected = state.editTargetId.length > 0;

  indicator.toggleClass?.("is-selected", selected);
  indicator.title = selected ? "Editing selected element" : "No selection";
}

async function editorTakeFocus() {
  if (shouldFocusEditor) myEditorView.focus()
}

async function canvaTakeFocus() {
  if (state.editTargetId.length > 0){
    const els = ea.getViewElements().filter(e => state.editTargetId.includes(e.id));
    if(els) ea.selectElementsInView(els);
    shouldFocusEditor = false // when canva take focus, we temporarily stop focusing the editor
  }
}

async function renderFullUI() {
  if(!globalContentEl) return;
  globalContentEl.empty();
  globalContentEl.addClass("excalimath-sidepanel");
  
  injectCSS(globalContentEl);
  
  const header = globalContentEl.createDiv({ cls: "excalimath-header" });
  header.innerHTML = `
  <h2>
    ${ea.obsidian.getIcon("radical").outerHTML}
    ExcaliMath
    <span class="excalimath-selection-indicator" title="No selection"></span>
  </h2>`;
  updateSelectionIndicator();
  if (!hasLatexSuite && state.activeTab === "formula") {
    const info = globalContentEl.createDiv({ cls: "excalimath-info" });
    info.innerHTML = `${ea.obsidian.getIcon("info").outerHTML} <span>${t("INFO_LATEX_SUITE")}</span>`;
  }
  
  const tabsContainer = globalContentEl.createDiv({ cls: "excalimath-tabs" });
  ["formula", "graph", "library"].forEach(tab => {
    const btn = tabsContainer.createEl("button", { text: t(`TAB_${tab.toUpperCase()}`) });
    if(state.activeTab === tab) btn.addClass("is-active");
    btn.onclick = () => {
      // If the user manually changes tabs, clear the edit target so the UI resets to "Insert"
      if (state.activeTab !== tab) {
        state.activeTab = tab;
        state.editTargetId = [];
        saveSettings();
        renderFullUI();
      }
    };
  });
  
  const tabContent = globalContentEl.createDiv({ cls: "excalimath-tab-content" });
  if(state.activeTab === "formula") await renderFormulaTab(tabContent);
  else if(state.activeTab === "graph") renderGraphTab(tabContent);
  else if(state.activeTab === "library") renderLibraryTab(tabContent);
}

function createPreviewArea(container) {
  const previewWrapper = container.createDiv({ cls: "excalimath-preview-wrapper" });
  previewWrapper.createDiv({ cls: "excalimath-preview-title", text: t("PREVIEW_TITLE") });
  const previewContainer = previewWrapper.createDiv({ cls: "excalimath-preview-container" });
  
  updatePreviewBackground(previewWrapper);
  return previewContainer;
}

function updatePreviewBackground(wrapper = null) {
  if (!wrapper) wrapper = globalContentEl?.querySelector(".excalimath-preview-wrapper");
  if (!wrapper) return;
  const st = ea.getExcalidrawAPI()?.getAppState();
  const isDark = st?.theme === "dark";
  const bgColor = st?.viewBackgroundColor || "#ffffff";
  
  wrapper.style.backgroundColor = bgColor;
  if (isDark) {
      wrapper.style.filter = "invert(93%) hue-rotate(180deg) saturate(1.25)";
  } else {
      wrapper.style.filter = "none";
  }
}

function updatePreviewArea(targetContainer = null) {
  const container = targetContainer || globalContentEl?.querySelector(".excalimath-preview-container");
  if(!container) return;
  
  clearTimeout(previewTimeout);
  previewTimeout = setTimeout(async () => {
    if(state.activeTab === "formula") await renderFormulaPreview(container);
    else if(state.activeTab === "graph") await renderGraphPreview(container);
  }, 300);
}

function updateButtonsColor(colorRow) {
  console.log(state.formula.color);
  const condition = (i) => COLOR_PRESETS[i].toLowerCase() === state.formula.color.toLowerCase();
    colorRow.querySelectorAll(".excalimath-color-btn").forEach((btn, i) => {
        btn.classList.toggle(
            "is-active",
            condition(i)
        );
    });
}

function updateButtonsFont(fontRow){
  const condition = (i) => FONT_PRESETS[i].wrapper === state.formula.font;
  fontRow.querySelectorAll(".excalimath-font-btn").forEach((btn, i) => {
      btn.classList.toggle(
          "is-active",
          condition(i)
      );
  });
}

function updateButtonsScale(scaleRow) {
  const condition = (i) => {
    let x = Math.abs(state.formula.scale - SCALE_PRESETS[i][1])
    return x < 0.01;
  }
  scaleRow.querySelectorAll("button").forEach((btn, i) => {
    btn.toggleClass(
      "is-active",
      condition(i)
    );
  });
}

// ---------------------------------------------------------
// 4. Formula Editor logic
// ---------------------------------------------------------
let renderTimeout = null;
let rescalingTimeout = null;
let recoloringTimeout = null;
let scaleTimeout = null;

const scheduleRender = () => {
  clearTimeout(renderTimeout);
  renderTimeout = setTimeout(() => {
    updateFormulae();
    saveSettings();
  }, 500); // wait after the last edit
};

const scheduleRecoloring = (colorRow) => {
  clearTimeout(recoloringTimeout);
  recoloringTimeout = setTimeout(() => {
    recoloring();
    updateButtonsColor(colorRow);
  }, 500);
}

const scheduleRescaling = (scaleRow) => {
  clearTimeout(rescalingTimeout);
  rescalingTimeout = setTimeout(() => {
    rescaling();
    updateButtonsScale(scaleRow);
  }, 500);
};


const scheduleScaleComputation = () => {
  clearTimeout(scaleTimeout);
  scaleTimeout = setTimeout(() => {
    updateScale();
    saveSettings();
  }, 500); // wait after the last edit
}


async function renderFormulaTabScale(container){
  const scaleSetting = new ea.obsidian.Setting(container)
    .setName(t("SCALE"))
    .setClass("excalimath-setting");
  const scaleRow = container.createDiv({ cls: "excalimath-scale-presets" });

  let scaleInput;

  scaleSetting.addText(text => {
    scaleInput = text;
    text
      .setValue(String(state.formula.scale || 1))
      .onChange(v => {
        const n = parseFloat(v);
        if (!isNaN(n) && n > 0) {
          state.formula.scale = n;
          scheduleRescaling(scaleRow);
        }
      });

    text.inputEl.style.width = "80px";
  });


  SCALE_PRESETS.forEach(([label, value]) => {
    const btn = scaleRow.createEl("button", {
      text: label,
      cls: "excalimath-scale-btn",
    });

    btn.onclick = () => {
      state.formula.scale = value;
      scaleInput.setValue(String(value));
      rescaling();
      updateButtonsScale(scaleRow);
    };
  });
  updateButtonsScale(scaleRow);
}

async function renderFormulaTabColor(container) {
  const colorSetting = new ea.obsidian.Setting(container)
    .setName(t("STROKE_COLOR"))
    .setClass("excalimath-setting");

  const colorRow = container.createDiv({
    cls: "excalimath-color-presets"
  });

  colorSetting.addText(text => {
      text
          .setPlaceholder("#1e1e1e")
          .setValue(state.formula.color)
          .onChange(v => {
              state.formula.color = v;
              recoloring();
              updateButtonsColor(colorRow);
              renderFullUI();
          });

      text.inputEl.style.width = "90px";
  });

  COLOR_PRESETS.forEach(color => {
      const btn = colorRow.createEl("button", {
          cls: "excalimath-color-btn"
      });

      btn.style.background = color;

      btn.onclick = () => {
          state.formula.color = color;
          recoloring();
          updateButtonsColor(colorRow);
          renderFullUI();
      };
  });

  // Color picker
  const picker = colorRow.createEl("input", {
      type: "color",
      cls: "excalimath-color-picker"
  });

  picker.value = state.formula.color;

  picker.oninput = () => {
      state.formula.color = picker.value;
      scheduleRecoloring(colorRow);
  };
  updateButtonsColor(colorRow);
}

async function renderFormulaTabFont(container) {
  const fontSetting = new ea.obsidian.Setting(container)
    .setName(t("FONT"))
    .setClass("excalimath-setting");

  const fontRow = container.createDiv({
    cls: "excalimath-font-presets"
  });

  FONT_PRESETS.forEach(({label, wrapper, html}) => {
      const btn = fontRow.createEl("button", {
          cls: "excalimath-font-btn"
      });
      
      btn.innerHTML = html;

      btn.onclick = () => {
          state.formula.font = wrapper;
          refont();
          updateButtonsFont(fontRow);
      };
  });
  updateButtonsFont(fontRow);
}

async function renderFormulaTab(container) {
  await renderFormulaTabColor(container);
  await renderFormulaTabFont(container);
  await renderFormulaTabScale(container);
  if (state.editTargetId.length === 0) {
    const newLatexBtn = container.createEl("button", {
      text: "+ new latex",
      cls: "excalimath-new-latex-btn",
    });
    newLatexBtn.onclick = () => newFormula();
  }
  if (state.editTargetId.length === 1) {
    container.appendChild(cmContainer);
  }
}

async function renderDynamicLibraryPreview(item, container) {
  ea.clear();
  let svg = null;
  try {
    if (item.type === "formula") {
      await ea.addLaTex(0, 0, item.data.text);
      const exportSettings = { withBackground: false, withTheme: false };
      svg = await ea.createSVG(null, false, exportSettings, undefined, "light", 10);
    } else {
      const elements = generateGraphElements(item.data.type, item.data.config);
      if (elements.length > 0) {
        elements.forEach(el => {
          ea.elementsDict[el.id] = el;
        });
        const exportSettings = { withBackground: false, withTheme: false };
        svg = await ea.createSVG(null, false, exportSettings, undefined, "light", 20);
      }
    }
  } catch(e) {
    console.error("ExcaliMath: Failed to render library preview", e);
  } finally {
    ea.clear();
  }

  if (svg) {
    svg.style.width = "100%";
    svg.style.height = "100%";
    svg.style.objectFit = "contain";
    container.appendChild(svg);
  } else {
    // Fallback to text description if SVG generation fails
    let desc = item.type === "formula" ? item.data.text : 
               (item.data.type === "custom" ? item.data.config.customFormula : item.data.type);
    container.createDiv({ text: desc, cls: "excalimath-lib-desc" });
  }
}

async function renderFormulaPreview(container) {
  container.empty();
  if(!state.formula.text || !hasMathJax) return;
  try {
    ea.clear();
    await ea.addLaTex(0, 0, await getFinalLatex(state.formula.text));
    
    // Generate an isolated SVG straight from the EA workbench without rendering back to the canvas
    const exportSettings = { withBackground: false, withTheme: false };
    const svg = await ea.createSVG(null, false, exportSettings, undefined, "light", 10);
    ea.clear();
    
    if(svg) {
      svg.style.width = "100%";
      svg.style.height = "100%";
      svg.style.objectFit = "contain";
      container.appendChild(svg);
    }
  } catch(e) {
    container.createEl("div", { text: t("INVALID_FORMULA") });
  }
}

async function updateScale(){
  if(!state.formula.text || !hasMathJax || state.editTargetId.length !== 1) return;
  const el = ea.getViewElements().find(e => e.id === state.editTargetId[0]);
  if (!el) return;
  const eq = ea.targetView.excalidrawData.getEquation(el.fileId);
  const scale = await getScale(el, eq.latex);
  if (Math.abs(state.formula.scale - scale) >= 0.01){
    state.formula.scale = scale;
    renderFullUI();
    saveSettings();
  }
}


//------------------
// the functions rescaling/recoloring/refont 
// apply also if there is multiple selected elements
// they all use changeSelectedElements as a template
//-------------------

/*
  map every selected elements using the function f
  delete the previous elements and add the new elements to view
*/
async function changeSelectedElements(f) {
  if(!state.formula.text || !hasMathJax) return;
  const newtarget = [];
  for (var id of state.editTargetId){
    let oldEl = ea.getViewElements().find(e => e.id === id);
    if (!oldEl) continue;
    let x = oldEl.x;
    let y = oldEl.y;
    let newEl = await f(oldEl);
    if(!newEl) continue;

    ea.copyViewElementsToEAforEditing([oldEl]);
    ea.getElement(oldEl.id).isDeleted = true;
    newEl.groupIds = [...oldEl.groupIds];
    newEl.angle = oldEl.angle;
    newtarget.push(newEl.id);
    // await ea.deleteViewElements([oldEl]);
  };
  await ea.addElementsToView(false, false, true);
  state.editTargetId = newtarget;
}

/*
apply the state.formula.scale to every selected elements
do not change the font or the color
*/
async function rescaling(){
  let f = async (oldEl) => {
    let x = oldEl.x;
    let y = oldEl.y;
    const scale = state.formula.scale || 1;
    const eq = ea.targetView.excalidrawData.getEquation(oldEl.fileId);
    if (!eq) return;
    const newid = await ea.addLaTex(x, y, eq.latex, scale, scale);
    return ea.getElement(newid);
  }
  changeSelectedElements(f);
}

/*
apply the state.formula.color to every selected elements
do not change the font or the scaling
*/
async function recoloring(){
  let f = async (oldEl) => {
    let x = oldEl.x;
    let y = oldEl.y;
    const eq = ea.targetView.excalidrawData.getEquation(oldEl.fileId);
    if (!eq) return;
    const scale = await getScale(oldEl, eq.latex);
    const latex = addColor(removeColor(eq.latex), state.formula.color);
    const newid = await ea.addLaTex(x, y, latex, scale, scale);
    return ea.getElement(newid);
  }
  changeSelectedElements(f);
}

/*
apply the state.formula.font to every selected elements
do not change the font or the scaling
*/
async function refont(){
  let f = async (oldEl) => {
    let x = oldEl.x;
    let y = oldEl.y;
    const eq = ea.targetView.excalidrawData.getEquation(oldEl.fileId);
    if (!eq) return;
    const color = await getColor(eq.latex);
    const latexempty = removeFont(removeColor(eq.latex));
    const scale = await getScale(oldEl, eq.latex);
    const latex = addColor(addFont(latexempty, state.formula.font), color);
    const newid = await ea.addLaTex(x, y, latex, scale, scale);
    return ea.getElement(newid);
  }
  changeSelectedElements(f);
}

async function updateFormulae(){
  if(!state.formula.text || !hasMathJax || state.editTargetId.length !== 1) return;
  let oldEl = ea.getViewElements().find(e => e.id === state.editTargetId[0]);
  if (!oldEl) return;
  let x = oldEl.x;
  let y = oldEl.y;
  let scale = state.formula.scale || 1;

  const id = await ea.addLaTex(x, y, await getFinalLatex(state.formula.text), scale, scale);
  const newEl = ea.getElement(id);

  ea.copyViewElementsToEAforEditing([oldEl]);
  ea.getElement(oldEl.id).isDeleted = true;
  newEl.groupIds = [...oldEl.groupIds];
  newEl.angle = oldEl.angle;
  state.editTargetId = [id];
  await ea.deleteViewElements([oldEl]);
  await ea.addElementsToView(false, false, true);
  
  // const finalEl = ea.getViewElements().find(e => e.id === id);
  // if(finalEl) ea.selectElementsInView([finalEl]);

}


async function newFormula() {
  if(!state.formula.text || !hasMathJax) return;
  ea.clear();
  let x = ea.getViewCenterPosition().x;
  let y = ea.getViewCenterPosition().y;
  
  // Default to the scale chosen in the UI dropdown (fallback to 1 "Small")
  let scale = state.formula.scale || 1;
  
  const id = await ea.addLaTex(x, y, await getFinalLatex(DEFAULT_LATEX), scale, scale);
  const newEl = ea.getElement(id);
  
  newEl.x -= newEl.width / 2;
  newEl.y -= newEl.height / 2;
  
  await ea.addElementsToView(false, false, true);
  const finalEl = ea.getViewElements().find(e => e.id === id);
  if(finalEl) ea.selectElementsInView([finalEl]);
  setTimeout(() => editorTakeFocus(), 100);
}

/*
switch focus to the editorView if there is one
create a new formula if there is none
does nothing if multiple elements are selected
*/
async function newFormulaOrFocus(){
  if (state.editTargetId.length > 1) return ;
  if (state.editTargetId.length === 1){
    editorTakeFocus()
    return;
  }else{
    newFormula();
  }

}

// ---------------------------------------------------------
// 5. Graph Editor logic
// ---------------------------------------------------------
function renderGraphTab(container) {
    
  const dynamicInputsContainer = container.createDiv();
  const type = state.graph.type;
  const activeParams = state.graphParams[type];
  

  new ea.obsidian.Setting(dynamicInputsContainer)
    .setName(t("GRAPH_FORMULA"))
    .setClass("excalimath-setting")
    .addText(text => {
      text.setValue(activeParams.customFormula);
      text.inputEl.style.width = "100%";
      text.inputEl.style.fontFamily = "monospace";
      text.onChange(v => { activeParams.customFormula = v; updatePreviewArea(); saveSettings(); });
    });

  const rangeSettingX = new ea.obsidian.Setting(container).setName(t("DOMAIN")).setClass("excalimath-setting");
  rangeSettingX.controlEl.addClass("excalimath-grid-2");
  rangeSettingX.addText(text => text.setValue(String(activeParams.xMin)).onChange(v => { activeParams.xMin = parseFloat(v); updatePreviewArea(); saveSettings(); }).inputEl.style.width="100%");
  rangeSettingX.addText(text => text.setValue(String(activeParams.xMax)).onChange(v => { activeParams.xMax = parseFloat(v); updatePreviewArea(); saveSettings(); }).inputEl.style.width="100%");
  
  const rangeSettingY = new ea.obsidian.Setting(container).setName(t("CODOMAIN")).setClass("excalimath-setting");
  rangeSettingY.controlEl.addClass("excalimath-grid-2");
  rangeSettingY.addText(text => text.setValue(String(activeParams.yMin)).onChange(v => { activeParams.yMin = parseFloat(v); updatePreviewArea(); saveSettings(); }).inputEl.style.width="100%");
  rangeSettingY.addText(text => text.setValue(String(activeParams.yMax)).onChange(v => { activeParams.yMax = parseFloat(v); updatePreviewArea(); saveSettings(); }).inputEl.style.width="100%");
  
  const colorSetting = new ea.obsidian.Setting(container).setName(t("STROKE_COLOR")).setClass("excalimath-setting");
  let textInput, colorPicker;
  colorSetting.addText(text => {
      textInput = text;
      text.setValue(activeParams.strokeColor).onChange(v => {
          activeParams.strokeColor = v;
          colorPicker.setValue(v);
          updatePreviewArea(); saveSettings();
      }).inputEl.style.width = "100px";
  });
  colorSetting.addColorPicker(picker => {
      colorPicker = picker;
      picker.setValue(activeParams.strokeColor).onChange(v => {
          activeParams.strokeColor = v;
          textInput.setValue(v);
          updatePreviewArea(); saveSettings();
      });
  });
  colorSetting.addButton(btn => btn
      .setIcon("swatch-book")
      .onClick(async () => {
          const selected = await ea.showColorPicker(btn.buttonEl, "elementStroke");
          if (selected) {
              activeParams.strokeColor = selected;
              textInput.setValue(selected);
              colorPicker.setValue(selected);
              updatePreviewArea(); saveSettings();
          }
      })
  );
  const bottomToggles = container.createDiv({ cls: "excalimath-grid-2" });

  new ea.obsidian.Setting(bottomToggles)
    .setName(t("SHOW_AXES"))
    .setClass("excalimath-setting")
    .addToggle(tgl => tgl.setValue(activeParams.showAxes).onChange(v => { activeParams.showAxes = v; renderFullUI(); saveSettings(); }));

  if (activeParams.showAxes) {
      const axesColorSetting = new ea.obsidian.Setting(container).setName(t("AXES_COLOR")).setClass("excalimath-setting");
      let axesTextInput, axesColorPicker;
      axesColorSetting.addText(text => {
          axesTextInput = text;
          text.setValue(activeParams.axesColor).onChange(v => {
              activeParams.axesColor = v;
              axesColorPicker.setValue(v);
              updatePreviewArea(); saveSettings();
          }).inputEl.style.width = "100px";
      });
      axesColorSetting.addColorPicker(picker => {
          axesColorPicker = picker;
          picker.setValue(activeParams.axesColor).onChange(v => {
              activeParams.axesColor = v;
              axesTextInput.setValue(v);
              updatePreviewArea(); saveSettings();
          });
      });
  }

  // --- Advanced options (collapsed by default; click header to expand) ---
  const advancedContainer = container.createDiv({ cls: "excalimath-advanced" });
  const advancedHeader = advancedContainer.createDiv({ cls: "excalimath-advanced-header" });
  advancedHeader.createSpan({ text: t("ADVANCED"), cls: "excalimath-advanced-title" });
  const advancedChevron = advancedHeader.createSpan({ cls: "excalimath-advanced-chevron", text: "▶" });
  const advancedBody = advancedContainer.createDiv({ cls: "excalimath-advanced-body" });

  const setAdvancedVisibility = (open) => {
    advancedBody.style.display = open ? "block" : "none";
    advancedChevron.setText(open ? "▼" : "▶");
    advancedHeader.toggleClass("is-active", open);
  };
  setAdvancedVisibility(state.graph.advancedOpen);

  advancedHeader.onclick = () => {
    state.graph.advancedOpen = !state.graph.advancedOpen;
    setAdvancedVisibility(state.graph.advancedOpen);
    saveSettings();
  };

  const scaleSetting = new ea.obsidian.Setting(advancedBody).setName(t("SCALE_XY")).setClass("excalimath-setting");
  scaleSetting.controlEl.addClass("excalimath-grid-2");
  scaleSetting.addText(text => text.setValue(String(activeParams.xScale)).onChange(v => { activeParams.xScale = parseFloat(v); updatePreviewArea(); saveSettings(); }).inputEl.style.width="100%");
  scaleSetting.addText(text => text.setValue(String(activeParams.yScale)).onChange(v => { activeParams.yScale = parseFloat(v); updatePreviewArea(); saveSettings(); }).inputEl.style.width="100%");

  new ea.obsidian.Setting(advancedBody)
    .setName(t("RESOLUTION"))
    .setClass("excalimath-setting")
    .addSlider(slider => slider.setLimits(10, 1000, 10).setValue(activeParams.resolution).onChange(v => { activeParams.resolution = v; updatePreviewArea(); saveSettings(); }));

  new ea.obsidian.Setting(advancedBody)
    .setName(t("STROKE_WIDTH"))
    .setClass("excalimath-setting")
    .addSlider(slider => slider.setLimits(0.5, 10, 0.5).setValue(activeParams.strokeWidth).onChange(v => { activeParams.strokeWidth = v; updatePreviewArea(); saveSettings(); }));

  new ea.obsidian.Setting(advancedBody)
    .setName(t("ROUGHNESS"))
    .setClass("excalimath-setting")
    .addSlider(slider => slider.setLimits(0, 2, 1).setValue(activeParams.roughness).onChange(v => { activeParams.roughness = v; updatePreviewArea(); saveSettings(); }));


  const previewContainer = createPreviewArea(container);
  updatePreviewArea(previewContainer);
}

function normalizePoints(points) {
  if(points.length === 0) return { offsetX: 0, offsetY: 0, points: [] };
  const minX = Math.min(...points.map(p=>p[0]));
  const minY = Math.min(...points.map(p=>p[1]));
  const normalized = points.map(p => [p[0] - minX, p[1] - minY]);
  return { offsetX: minX, offsetY: minY, points: normalized };
}

function createRawLineElement(points, color, width, roughness, isPolygon) {
  const norm = normalizePoints(points);
  const w = Math.abs(Math.max(...norm.points.map(p=>p[0])) - Math.min(...norm.points.map(p=>p[0])));
  const h = Math.abs(Math.max(...norm.points.map(p=>p[1])) - Math.min(...norm.points.map(p=>p[1])));
  
  return {
    id: ea.generateElementId(),
    type: "line",
    x: norm.offsetX,
    y: norm.offsetY,
    width: w,
    height: h,
    angle: 0,
    strokeColor: color || "#1e1e1e",
    backgroundColor: "transparent",
    fillStyle: "solid",
    strokeWidth: width || 2,
    strokeStyle: "solid",
    roughness: roughness || 0,
    opacity: 100,
    groupIds: [],
    frameId: null,
    roundness: null,
    seed: Math.floor(Math.random() * 100000),
    version: 1,
    versionNonce: Math.floor(Math.random() * 100000),
    isDeleted: false,
    boundElements: null,
    updated: Date.now(),
    link: null,
    locked: false,
    points: norm.points,
    lastCommittedPoint: null,
    startBinding: null,
    endBinding: null,
    startArrowhead: null,
    endArrowhead: null,
    polygon: isPolygon
  };
}

//---------------
//  Parsing Custom function
//---------------
// the goal is to go from a user typed function 
// to a TS valid function for this we add `Math.*` where needed


function generateGraphPoints(type, graphConfig) {
  const points = [];
  const step = (graphConfig.xMax - graphConfig.xMin) / (Math.max(1, graphConfig.resolution - 1));
  try {
    if(!window.mathGraphCustomFunc || window.mathGraphCustomFuncString !== graphConfig.customFormula) {
      window.mathGraphCustomFunc = new Function('x', `with (Math) { return ${graphConfig.customFormula}; }`);
      window.mathGraphCustomFuncString = graphConfig.customFormula;
    }
  } catch(e) {
    return [];
  }

  for(let i=0; i < graphConfig.resolution; i++) {
    const x = graphConfig.xMin + i * step;
    let y = 0;
    try {
      y = window.mathGraphCustomFunc(x);
    }catch(e){
      continue;
    }
    if(isNaN(y) || !isFinite(y)) continue;
    
    const px = x * graphConfig.xScale;
    const py = -y * graphConfig.yScale;
    points.push([px, py]);
  }
  if(graphConfig.closePlot && points.length > 0) {
    points.push([points[0][0], points[0][1]]);
  }
  return points;
}

function generateGraphElements(type, graphConfig) {
  const points = generateGraphPoints(type, graphConfig);
  if(points.length < 2) return [];

  const elements = [];

  if (graphConfig.showAxes) {
    const xs = points.map(p => p[0]);
    const ys = points.map(p => p[1]);
    
    const minX = Math.min(...xs, 0);
    const maxX = Math.max(...xs, 0);
    const minY = Math.min(...ys, 0);
    const maxY = Math.max(...ys, 0);
    
    const xPad = Math.max(Math.abs(maxX - minX) * 0.05, 20);
    const yPad = Math.max(Math.abs(maxY - minY) * 0.05, 20);

    const xAxis = createRawLineElement([[minX - xPad, 0], [maxX + xPad, 0]], graphConfig.axesColor || "#888888", 1, 0, false);
    const yAxis = createRawLineElement([[0, minY - yPad], [0, maxY + yPad]], graphConfig.axesColor || "#888888", 1, 0, false);
    elements.push(xAxis, yAxis);
  }

  const mainLine = createRawLineElement(points, graphConfig.strokeColor, graphConfig.strokeWidth, graphConfig.roughness, graphConfig.closePlot);
  mainLine.customData = { mathGraph: { type, config: { ...graphConfig } } };
  elements.push(mainLine);

  return elements;
}

async function getPreviewDataURL(tabType, data) {
  ea.clear();
  
  if (tabType === "formula") {
     await ea.addLaTex(0, 0, data.text);
  } else {
     const elements = generateGraphElements(data.type, data.config);
     if (elements.length === 0) return null;
     
     // Inject newly generated elements into the workbench mapping
     elements.forEach(el => {
        ea.elementsDict[el.id] = el;
     });
  }

  // Create SVG exclusively from the workbench contents
  const exportSettings = { withBackground: false, withTheme: false };
  const svg = await ea.createSVG(null, false, exportSettings, undefined, "light", 20);
  ea.clear();
  
  if (!svg) return null;
  
  // Convert the exported SVG to a fully formed Base64 Data URL for the Library `img.src`
  const svgString = svg.outerHTML;
  const base64 = btoa(unescape(encodeURIComponent(svgString)));
  return "data:image/svg+xml;base64," + base64;
}

async function renderGraphPreview(container) {
  container.empty();
  ea.clear();
  const type = state.graph.type;
  const config = state.graphParams[type];
  
  const elements = generateGraphElements(type, config);
  if(elements.length === 0) {
    container.createEl("div", { text: t("INVALID_FORMULA") });
    return;
  }
  
  // Push elements into the EA workbench manually
  elements.forEach(el => {
     ea.elementsDict[el.id] = el;
  });
  
  const exportSettings = { withBackground: false, withTheme: false };
  const svg = await ea.createSVG(null, false, exportSettings, undefined, "light", 20);
  ea.clear();
  
  if(svg) {
      svg.style.width = "100%";
      svg.style.height = "100%";
      svg.style.objectFit = "contain";
      container.appendChild(svg);
  }
}

async function insertGraph() {
  ea.clear();
  const type = state.graph.type;
  const config = state.graphParams[type];
  const elementsData = generateGraphElements(type, config);
  
  if(elementsData.length === 0) {
    new Notice(t("INVALID_FORMULA"));
    return;
  }
  
  let x = ea.getViewCenterPosition().x;
  let y = ea.getViewCenterPosition().y;
  let oldEl = null;
  
  if (state.editTargetId) {
    oldEl = ea.getViewElements().find(e => e.id === state.editTargetId || (e.groupIds && e.groupIds.includes(state.editTargetId)));
    if (oldEl) {
      const groupToDel = oldEl.groupIds?.length ? ea.getViewElements().filter(e => e.groupIds.includes(oldEl.groupIds[0])) : [oldEl];
      
      const bb = ea.getBoundingBox(groupToDel);
      x = bb.topX + bb.width / 2;
      y = bb.topY + bb.height / 2;

      ea.copyViewElementsToEAforEditing(groupToDel);
      ea.getElements().forEach(e => e.isDeleted = true);
    }
  }
  
  const ids = [];
  elementsData.forEach(elData => {
    ea.elementsDict[elData.id] = elData;
    ids.push(elData.id);
  });
  
  if (ids.length > 1) {
    ea.addToGroup(ids);
  }
  
  const bb = ea.getBoundingBox(ids.map(id => ea.getElement(id)));
  
  const dx = x - (bb.width/2) - bb.topX;
  const dy = y - (bb.height/2) - bb.topY;
  ids.forEach(id => {
    const e = ea.getElement(id);
    e.x += dx; e.y += dy;
  });
  
  await ea.addElementsToView(false, false, true);
  const finalEls = ea.getViewElements().filter(e => ids.includes(e.id));
  if(finalEls.length > 0) ea.selectElementsInView(finalEls);
}

// ---------------------------------------------------------
// 6. Library logic
// ---------------------------------------------------------
function renderLibraryTab(container) {
  if (state.library.length === 0) {
    container.createEl("p", { text: t("LIBRARY_EMPTY"), cls: "excalimath-info" });
    return;
  }
  
  state.library.forEach((item, idx) => {
    const card = container.createDiv({ cls: "excalimath-lib-card" });
    card.innerHTML = `<strong>${ea.obsidian.getIcon(item.type === "formula" ? "radical" : "trending-up").outerHTML} ${item.name}</strong>`;
    
    const previewDiv = card.createDiv({ cls: "excalimath-lib-preview" });
    
    const st = ea.getExcalidrawAPI()?.getAppState();
    const isDark = st?.theme === "dark";
    const bgColor = st?.viewBackgroundColor === "transparent" 
        ? (isDark ? "#1e1e1e" : "#ffffff") 
        : (st?.viewBackgroundColor || "#ffffff");
        
    previewDiv.style.backgroundColor = bgColor;
    if (isDark) {
        previewDiv.style.filter = "invert(93%) hue-rotate(180deg) saturate(1.25)";
    }

    // Render the preview dynamically on the fly
    renderDynamicLibraryPreview(item, previewDiv);

    const actions = card.createDiv({ cls: "excalimath-lib-actions" });
    
    const delBtn = actions.createEl("button", { text: t("DELETE") });
    delBtn.innerHTML = `${ea.obsidian.getIcon("trash").outerHTML} ${t("DELETE")}`;
    delBtn.style.color = "var(--text-error)";
    delBtn.onclick = () => {
      state.library.splice(idx, 1);
      saveSettings();
      renderFullUI();
    };
    
    const loadBtn = actions.createEl("button", { text: t("LOAD"), cls: "mod-cta" });
    loadBtn.innerHTML = `${ea.obsidian.getIcon("upload").outerHTML} ${t("LOAD")}`;
    loadBtn.onclick = () => {
      if(item.type === "formula") {
        state.formula = { ...state.formula, ...item.data };
        state.activeTab = "formula";
      } else {
        state.graph.type = item.data.type;
        state.graphParams[item.data.type] = { ...state.graphParams[item.data.type], ...item.data.config };
        state.activeTab = "graph";
      }
      saveSettings();
      renderFullUI();
    };
  });
}

async function addToLibrary(type, data) {
  const name = await utils.inputPrompt(t("PROMPT_NAME"), "", `My ${type}`);
  if(!name) return;
  
  // Exclusively store the configuration parameters. No bloated data URLs.
  state.library.push({
    id: ea.generateElementId(),
    name,
    type,
    data: JSON.parse(JSON.stringify(data))
  });
  
  await saveSettings();
  new Notice(`Saved to library: ${name}`);
  renderFullUI();
}

// ---------------------------------------------------------
// 7. Event Hooks & Setup
// ---------------------------------------------------------
/* return the color if they all have the same, else return #000000*/
async function agreedColor(editTargetId, selectedId){
  return "#000000"
}

/* return the font if they all have the same, else return "error"*/
async function agreedFont(editTargetId, selectedId){
  return "error"
}

/* return the scale if they all have the same, else return 0*/
async function agreedScale(editTargetId, selectedId){
  return 0
}

async function selectSingleLatex(selected) {
  const el = selected[0];
  const eq = ea.targetView.excalidrawData.getEquation(el.fileId);
  const latexnoColor = removeColor(eq.latex);
  const latexfinal = removeFont(latexnoColor);
  if(!eq) return;
  state.editTargetId = [el.id];
  state.activeTab = "formula";
  state.formula.text = latexfinal;
  if (!myEditorView) return;
  const doc = myEditorView.state.doc;
  myEditorView.dispatch({
    changes: {
      from: 0,
      to: doc.length,
      insert: latexfinal,
    },
    selection: {
      anchor: 0,
      head: latexfinal.length,
    },
  });
  if (shouldFocusEditor) {
    setTimeout( () => myEditorView.focus(), 100);
  }else{
    shouldFocusEditor = true; 
    // next time we can focus the editor back
  }
  state.formula.color = getColor(eq.latex);
  state.formula.font = getFont(latexnoColor);
  state.formula.scale = await getScale(el, eq.latex);
}

async function selectMultiple(selected){
  state.selectedId =  // select all non-latex elements
    selected
      .filter((el) => {
        const equation = ea.targetView.excalidrawData.getEquation(el.fileId); 
        return el.type !== "image" || !equation })
      .map((el) => el.id);
  state.editTargetId =  //select all latex elements
    selected
      .filter((el)=> {
        const equation = ea.targetView.excalidrawData.getEquation(el.fileId); 
        return el.type === "image" && !!equation })
      .map((el) => el.id);
  console.log("multiple?");
  state.formula.color = await agreedColor(state.editTargetId, state.selectedId);
  state.formula.scale = await agreedScale(state.editTargetId, state.selectedId);
  state.formula.font = await agreedFont(state.editTargetId, state.selectedId);
}

async function selectGraph(selected){
  state.editTargetId = [mathGraphEl.id];
  state.activeTab = "graph";
  
  const md = mathGraphEl.customData.mathGraph;
  if (md.type) {
      state.graph.type = md.type;
      state.graphParams[md.type] = { ...state.graphParams[md.type], ...md.config };
  } else {
      // Backwards compatibility for older saved graphs
      state.graph.type = "custom";
      state.graphParams.custom = { ...state.graphParams.custom, ...md };
  }
}

async function checkSelection() {
  const selected = await ea.getViewSelectedElements();
  let changed = false;
  if (selected.length === 0){
    state.editTargetId = [];
    changed = true;
  }
  if(selected.length === 1) {
    let mathGraphEl = selected.find(el => el.customData?.mathGraph);
    if (!mathGraphEl && selected[0].groupIds?.length > 0) {
       const groupElements = ea.getViewElements().filter(e => e.groupIds.includes(selected[0].groupIds[0]));
       mathGraphEl = groupElements.find(e => e.customData?.mathGraph);
    }
    
    if (mathGraphEl && state.editTargetId !== mathGraphEl.id) {
      selectGraph(selected);
      changed = true;
    } else if (selected[0].type === "image") {
      selectSingleLatex(selected);
      changed = true;
    } else {
      state.editTargetId = [];
      changed = true;
    }
  }
  if (selected.length > 1) {
    selectMultiple(selected);
    changed = true;
  }

  if (changed) {
    saveSettings();
    renderFullUI();
  }
  return changed;
}

// ---------------------------------------------------------
// Main Execution
// ---------------------------------------------------------
async function main() {
  if (!hasMathJax) {
    new Notice(t("ERROR_MATHJAX"));
    return;
  }

  const existingTab = ea.checkForActiveSidepanelTabForScript();
  if (existingTab) {
    const hostEA = existingTab.getHostEA();
    if (hostEA && hostEA !== ea) {
      hostEA.setView(ea.targetView);
      existingTab.open();
      return;
    }
  }

  await loadSettings();

  const tab = await ea.createSidepanelTab("ExcaliMath", true, true);
  if (!tab) return;

  globalContentEl = tab.contentEl;
  createEditorView();
  tab.onOpen = () => {
    renderFullUI();
    checkSelection();
  };

  tab.onFocus = (view) => {
    if (view && view !== ea.targetView) {
      ea.setView(view);
      ea.clear();
    }
    updatePreviewBackground();
    if (state.activeTab !== "library") {
       updatePreviewArea();
    }
  //  checkSelection();

    // Auto-focus the formula editor whenever the sidepanel receives focus
    // if (state.activeTab === "formula") {
    //   if (window.excalimathEditorView) {
    //     window.excalimathEditorView.focus();
    //   } else {
    //     globalContentEl?.querySelector('textarea')?.focus();
    //   }
    // }
  };

  // const keymapScope = app.keymap.scope;
  // const keymaphandler = keymapScope.register(["Alt"], "KeyL", (e) => newFormulaOrFocus());

  tab.onClose = () => {
    ea.onSceneChangeHook = null;
    saveSettings();
    // keymapScope.unregister(keymapScope);
    delete window.excalimathEditorView;
  };

  ea.onSceneChangeHook = {
    appStateKeys: ["selectedElementIds", "newElement", "theme", "viewBackgroundColor", "isResizing"],
    trackElements: false,
    triggerWhenInvisible: false,
    callback: (elements, appState, files, view, hookEA) => {
      if (view && view !== ea.targetView) {
        ea.setView(view);
        ea.clear();
      }
      let needsPreviewUpdate = false;
      const st = appState;

      const currentBgColor = st.viewBackgroundColor === "transparent" 
          ? (st.theme === "dark" ? "#1e1e1e" : "#ffffff") 
          : (st.viewBackgroundColor || "#ffffff");
          
      const isDark = st.theme === "dark";
      const wrapper = globalContentEl?.querySelector(".excalimath-preview-wrapper");
      
      if (wrapper) {
         if (wrapper.style.backgroundColor !== currentBgColor || 
            (isDark && wrapper.style.filter === "none") ||
            (!isDark && wrapper.style.filter !== "none")
         ) {
             updatePreviewBackground(wrapper);
         }
      }
      //scheduleScaleComputation();
      checkSelection();
    }
  };

  tab.open();
  
}

main();