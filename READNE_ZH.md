# 使用方法
### 进入官网解密界面：exspectamus.dontstarvetogether.com；    

### 打开开发人员工具：在饥荒解谜页面，按键盘上的 F12 键 或 Ctrl + Shift + I 打开开发者工；

### 打开控制台：点击 Console / 控制台 标签，打开控制台；

### 加载脚本到页面中：滚动到下方找到完整代码框，全选代码后按 Ctrl + C 复制，回到控制台，点击光标，Ctrl + V 粘贴代码，按回车键 (Enter)加载。

### 常用指令：

auto() —— 全自动往后执行所有

next() —— 单步执行

list() —— 查看所有解谜步骤

jumpTo(n) —— 直接跳到第 n 步

stop() —— 紧急暂停脚本

注意：模拟拖拽时不要使用鼠标滚轮移动页面否则物品会乱飞

```
// ==================== 饥荒自动解谜 - 终极整合版 ====================
(function() {
'use strict';

// ==================== 页面自动寻路系统 ====================

// 定义页面间的跳转关系和点击位置
const PAGE_TRANSITIONS = {
    '0': {
        '1':  { panel: 0, x: 17, y: 15 }, // 太阳切黑夜
        '78': { panel: 1, x: 14, y: 77 }  // 铅笔切书房
    },
    '1': {
        '0':  { panel: 0, x: 17, y: 15 }, // 月亮切白天
        '4':  { panel: 0, x: 53, y: 85 }, // 土坑切废墟
        '2':  { panel: 16, x: 54, y: 39 } // 音乐之门纸条切修理室
    },
    '2': {
        '78': { panel: 0, x: 54, y: 44 }, // 蜡笔切书房
        '5':  { panel: 0, x: 22, y: 46 }  // 门切透镜房
    },
    '4': {
        '2':  { panel: 0, x: 21, y: 70 }  // 老瓦零件切修理室
    },
    '5': {
        '2':  { panel: 0, x: 22, y: 46 }  // 门切修理室
    },
    '78': {
        '1':  { panel: 0, x: 42, y: 89 }, // 硬币切黑夜
        '2':  { panel: 0, x: 43, y: 62 }, // 螺丝刀切修理室
        '4':  { panel: 0, x: 13, y: 32 }  // 零件切废墟
    }
};

function getCurrentPage() {
    return document.body.getAttribute('page') || '0'; // 找不到默认认为在白天(0)
}

// 核心：基于 BFS 查找最短跳转路径并自动点击
async function navigateToPage(targetPage) {
    targetPage = String(targetPage);
    let currentPage = getCurrentPage();

    if (currentPage === targetPage) return;

    console.log(`\n🚗 自动寻路：当前在 Page ${currentPage}，目标 Page ${targetPage}`);

    let queue = [ { page: currentPage, path: [] } ];
    let visited = new Set([currentPage]);
    let foundPath = null;

    while (queue.length > 0) {
        let current = queue.shift();
        if (current.page === targetPage) {
            foundPath = current.path;
            break;
        }

        let neighbors = PAGE_TRANSITIONS[current.page];
        if (neighbors) {
            for (let nextPage in neighbors) {
                if (!visited.has(nextPage)) {
                    visited.add(nextPage);
                    queue.push({
                        page: nextPage,
                        path: [...current.path, { from: current.page, to: nextPage, action: neighbors[nextPage] }]
                    });
                }
            }
        }
    }

    if (!foundPath) {
        console.error(`❌ 无法找到从页面 ${currentPage} 到页面 ${targetPage} 的路径！你可能处于未知状态。`);
        return;
    }

    // 沿路径执行跳转
    for (let i = 0; i < foundPath.length; i++) {
        let step = foundPath[i];
        console.log(`   [跳页 ${i + 1}/${foundPath.length}] 从 ${step.from} 前往 ${step.to}`);

        await simulateClick({
            page: step.from,
            panel: step.action.panel,
            x: step.action.x,
            y: step.action.y,
            wait: 3500, // 给予跳页动画加载的时间
            desc: `跳页路线 -> ${step.to}`,
            skipNav: true
        });

        let newPage = getCurrentPage();
        if (newPage !== step.to) {
             console.warn(`   ⚠️ 页面似乎没有更新，当前依然是 ${newPage}。可能在加载中或前置条件未解锁。`);
        }
    }
    console.log(`✅ 寻路到达目标 Page ${getCurrentPage()}\n`);
}

// ==================== 工具与反馈函数 ====================

function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

async function waitForElement(selector, timeout) {
    if (timeout === undefined) {
        // 特殊判断：#page1panel6 需要长达三分钟的超时来等待蟋蟀出现
        if (selector === '#page1panel6') {
            timeout = 180000;
            console.log(`⏳ [提示] 等待蟋蟀/小卷两分钟...请耐心等待页面刷新`);
        } else {
            timeout = 8000;
        }
    }
    const start = Date.now();
    while (Date.now() - start < timeout) {
        const panelContainer = document.querySelector(selector);
        if (panelContainer) {
            const clickTarget = panelContainer.querySelector('.panel-background');
            if (clickTarget) return clickTarget;
        }
        await sleep(500);
    }
    throw new Error(`元素未找到: ${selector} (等待了 ${timeout / 1000} 秒)`);
}

async function waitForReady(timeout = 10000) {
    const start = Date.now();
    while (Date.now() - start < timeout) {
        if (!document.body.classList.contains('loading')) {
            await sleep(200);
            return true;
        }
        await sleep(100);
    }
    return false;
}

function highlightClick(x, y) {
    const dot = document.createElement('div');
    dot.style.cssText = `position:fixed;left:${x}px;top:${y}px;width:20px;height:20px;margin:-10px 0 0 -10px;border-radius:50%;background:rgba(255,0,0,0.5);border:2px solid red;pointer-events:none;z-index:999999;animation:pulse 0.5s ease-out;`;
    document.body.appendChild(dot);
    setTimeout(() => dot.remove(), 500);
}

function highlightDrag(startX, startY, endX, endY) {
    const line = document.createElement('div');
    const length = Math.sqrt(Math.pow(endX - startX, 2) + Math.pow(endY - startY, 2));
    const angle = Math.atan2(endY - startY, endX - startX) * 180 / Math.PI;
    line.style.cssText = `position:fixed;left:${startX}px;top:${startY}px;width:${length}px;height:3px;background:linear-gradient(to right,rgba(0,100,255,0.8),rgba(0,200,255,0.8));transform-origin:0 50%;transform:rotate(${angle}deg);pointer-events:none;z-index:999999;animation:dragFade 1s ease-out;`;
    document.body.appendChild(line);
    setTimeout(() => line.remove(), 1000);
}

const style = document.createElement('style');
style.textContent = `@keyframes pulse { 0% { transform: scale(0.5); opacity: 1; } 100% { transform: scale(2); opacity: 0; } } @keyframes dragFade { 0% { opacity: 1; } 70% { opacity: 1; } 100% { opacity: 0; } }`;
document.head.appendChild(style);

// ==================== 智能防错守护系统 ====================

async function checkAndClosePanel(page, panel) {
    const pg = String(page);
    const pn = String(panel);

    if (pg === '78' && pn === '0') {
        let img = document.querySelector('#page78panel0 img.inspectable.open');
        if (img) {
            console.log("  => [防错保护] 检测到 page78panel0 存在未关闭的物品，执行复位...");
            await simulateClick({page: 78, panel: 0, x: 0, y: 0, wait: 1000, skipNav: true, skipCheck: true});
        }
    } else if (pg === '0' && pn === '1') {
        let img = document.querySelector('#page0panel1 img.inspectable.open');
        if (img) {
            console.log("  => [防错保护] 检测到 page0panel1 存在未关闭的物品，执行复位...");
            await simulateClick({page: 0, panel: 1, x: 0, y: 0, wait: 1000, skipNav: true, skipCheck: true});
        }
    } else if (pg === '5' && pn === '1') {
        if (document.getElementById('page5panel1openbook')) {
            console.log("  => [防错保护] 检测到 page5panel1 存在未关闭的书，执行复位...");
            await simulateClick({page: 5, panel: 1, x: 0, y: 0, wait: 1000, skipNav: true, skipCheck: true});
        }
    }
}

// ==================== 动作执行引擎 ====================

async function simulateClick({page, panel, x, y, wait = 1000, desc = '', skipNav = false, skipCheck = false}) {
    if (!skipNav && page !== undefined) {
        await navigateToPage(page);
    }

    // 在面板点击前，进行智能闭合物品检测
    if (!skipCheck && page !== undefined && panel !== undefined) {
        await checkAndClosePanel(page, panel);
    }

    const selector = `#page${page}panel${panel}`;
    const element = await waitForElement(selector);
    const rect = element.getBoundingClientRect();
    const clientX = rect.left + rect.width * x / 100;
    const clientY = rect.top + rect.height * y / 100;

    highlightClick(clientX, clientY);

    element.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true, clientX: clientX, clientY: clientY, view: window }));
    console.log(`  ✓ Panel ${panel} (${x}%, ${y}%)${desc ? ` (${desc})` : ''}`);

    await waitForReady();
    await sleep(wait);
}

async function simulateDrag({page, sliderId, startCoord, endCoord, valueChange, wait = 1500, skipNav = false}) {
    if (!skipNav && page !== undefined) {
        await navigateToPage(page);
    }

    const element = document.getElementById(sliderId);
    if (!element) return console.error(`  ✗ 元素未找到: ${sliderId}`);

    const isDraggablePanel = element.hasAttribute('candrag') && element.getAttribute('candrag') === 'true';
    const isDial = element.tagName.toLowerCase() === 'goggles-dial';
    const isMonocle = element.tagName.toLowerCase() === 'shadow-monocle';

    if (isDraggablePanel || isDial || isMonocle) {
        console.log(`  🔄 拖拽 ${sliderId}`);
        const parentRect = element.parentElement.getBoundingClientRect();
        let startX, startY;

        if (startCoord === 'auto') {
            const rect = element.getBoundingClientRect();
            startX = rect.left + rect.width / 2;
            startY = rect.top + rect.height / 2;
        } else {
            startX = parentRect.left + parentRect.width * startCoord.x / 100;
            startY = parentRect.top + parentRect.height * startCoord.y / 100;
        }

        const endX = parentRect.left + parentRect.width * endCoord.x / 100;
        const endY = parentRect.top + parentRect.height * endCoord.y / 100;
        highlightDrag(startX, startY, endX, endY);

        const targetNode = isDial ? (element.querySelector('.dial-handle') || element) : element;
        targetNode.dispatchEvent(new PointerEvent('pointerdown', { bubbles:true, clientX:startX, clientY:startY, isPrimary:true }));
        targetNode.dispatchEvent(new MouseEvent('mousedown', { bubbles:true, clientX:startX, clientY:startY }));
        await sleep(50);

        targetNode.dispatchEvent(new PointerEvent('pointermove', { bubbles:true, clientX:endX, clientY:endY, isPrimary:true }));
        targetNode.dispatchEvent(new MouseEvent('mousemove', { bubbles:true, clientX:endX, clientY:endY }));
        await sleep(100);

        targetNode.dispatchEvent(new PointerEvent('pointerup', { bubbles:true, clientX:endX, clientY:endY, isPrimary:true }));
        targetNode.dispatchEvent(new MouseEvent('mouseup', { bubbles:true, clientX:endX, clientY:endY }));
    } else {
        console.log(`  🔄 滑块 ${sliderId}`);
        const back = element.querySelector('.rangeslider-back');
        const knob = element.querySelector('.rangeslider-knob');
        if (!back || !knob) return console.error(`  ✗ 滑块组件缺失结构`);

        const sliderRect = element.getBoundingClientRect();
        const backRect = back.getBoundingClientRect();
        const startY = sliderRect.top + sliderRect.height * startCoord.y / 100;
        const endY = sliderRect.top + sliderRect.height * endCoord.y / 100;
        const centerX = backRect.left + backRect.width / 2;
        highlightDrag(centerX, startY, centerX, endY);

        back.dispatchEvent(new PointerEvent('pointerdown', { bubbles:true, clientX:centerX, clientY:startY, isPrimary:true }));
        back.dispatchEvent(new MouseEvent('mousedown', { bubbles:true, clientX:centerX, clientY:startY }));
        await sleep(50);

        back.dispatchEvent(new PointerEvent('pointermove', { bubbles:true, clientX:centerX, clientY:endY, isPrimary:true }));
        back.dispatchEvent(new MouseEvent('mousemove', { bubbles:true, clientX:centerX, clientY:endY }));
        await sleep(100);

        back.dispatchEvent(new PointerEvent('pointerup', { bubbles:true, clientX:centerX, clientY:endY, isPrimary:true }));
        back.dispatchEvent(new MouseEvent('mouseup', { bubbles:true, clientX:centerX, clientY:endY }));
    }
    await waitForReady();
    await sleep(wait);
}

// ==================== 智能配药助手函数 ====================

async function smartPreparePotion(recipeIds) {
    if (getCurrentPage() !== '1') await navigateToPage('1');

    // 智能药罐检测
    let panel20 = document.getElementById('page1panel20');
    if (!panel20) {
        console.log("  => 正在进入药罐界面...");
        await simulateClick({page: 1, panel: 16, x: 85, y: 62, wait: 5000, skipNav: true});
    }

    // 判断是否需要舀树液（如果不存在 984 液体，说明器皿是空的）
    if (!document.getElementById('page1panel20-984')) {
        console.log("  => [智能初始化] 检测到器皿缺树液...");

        // 找空勺子去舀
        let emptySpoon = document.getElementById('page1panel20-477');
        if (emptySpoon) {
            await simulateDrag({page: 1, sliderId: 'page1panel20-477', startCoord: 'auto', endCoord: {x: 92, y: 67}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true});
        }

        // 把装满的勺子倒进器皿
        let fullSpoonId = document.getElementById('page1panel20-765') ? 'page1panel20-765' : 'page1panel20-477';
        await simulateDrag({page: 1, sliderId: fullSpoonId, startCoord: 'auto', endCoord: {x: 47, y: 43}, valueChange: {from: 0, to: 1}, wait: 3000, skipNav: true});
    }

    // 动态辅助采集方法 (灌木丛里的虫子)
    const waitAndClickBug = async (targetX, targetY) => {
        let panel6 = document.getElementById('page1panel6');
        if (!panel6) {
            console.log("  => [辅助] 正在点击灌木丛，需耐心等待虫子爬出...");
            await simulateClick({page: 1, panel: 7, x: 9, y: 70, wait: 2000, skipNav: true});
            await waitForElement('#page1panel6'); // 等待面板6出现
        }
        await simulateClick({page: 1, panel: 6, x: targetX, y: targetY, wait: 2000, skipNav: true});
    };

    // 字典定义所有材料、采集方法及【中心点初始位置】
    const INGREDIENTS = {
        'page1panel20-948': { name: '蘑菇(橘白)', start: {x: 5.4, y: 66.85}, get: async () => { await simulateClick({page: 1, panel: 8, x: 36, y: 82, wait: 2000, skipNav: true}); } },
        'page1panel20-908': { name: '树莓', start: {x: 29.4, y: 47.95}, get: async () => { await simulateClick({page: 1, panel: 4, x: 51, y: 52, wait: 2000, skipNav: true}); } },
        'page1panel20-712': { name: '蘑菇(黄卷)', start: {x: 20.3, y: 42.8}, get: async () => {
            await simulateClick({page: 1, panel: 7, x: 20, y: 80, wait: 2000, skipNav: true});
            await simulateClick({page: 1, panel: 7, x: 37, y: 70, wait: 2000, skipNav: true});
        } },
        'page1panel20-432': { name: '蟋蟀', start: {x: 44.5, y: 66.45}, get: async () => { await waitAndClickBug(26, 77); } },
        'page1panel20-378': { name: '蘑菇(红伞)', start: {x: 83.1, y: 72.6}, get: async () => { await simulateClick({page: 1, panel: 7, x: 80, y: 81, wait: 2000, skipNav: true}); } },
        'page1panel20-876': { name: '蘑菇(小卷)', start: {x: 69.9, y: 62.85}, get: async () => { await waitAndClickBug(83, 84); } },
        'page1panel20-547': { name: '蓝莓', start: {x: 24.55, y: 59.6}, get: async () => { await simulateClick({page: 1, panel: 12, x: 92, y: 12, wait: 2000, skipNav: true}); } },
        'page1panel20-602': { name: '蜜蜂', start: {x: 77.05, y: 35.7}, get: async () => { await simulateClick({page: 1, panel: 10, x: 65, y: 63, wait: 2000, skipNav: true}); } },
        'page1panel20-496': { name: '蚯蚓', start: {x: 5.6, y: 84.35}, get: async () => { await simulateClick({page: 1, panel: 8, x: 13, y: 82, wait: 2000, skipNav: true}); } },
    };

    // 2. 检查屏幕上是否存在所需材料，若缺少则跳出执行采集
    for (let id of recipeIds) {
        if (!document.getElementById(id)) {
            console.log(`  => ⚠ 发现缺少配方材料 [${INGREDIENTS[id].name}]，自动前往采集...`);
            await navigateToPage('1');
            await INGREDIENTS[id].get();
            console.log(`  => [${INGREDIENTS[id].name}] 采集完成，返回药罐...`);
            await navigateToPage('1');
            await simulateClick({page: 1, panel: 16, x: 85, y: 62, wait: 4000, skipNav: true});
        }
    }

    // 3. 将碾子复位 (防挡)
    let pestle = document.getElementById('page1panel20-271');
    if (pestle) {
        let px = (parseFloat(pestle.style.left) || 0) + (parseFloat(pestle.style.width) || 0) / 2;
        let py = (parseFloat(pestle.style.top) || 0) + (parseFloat(pestle.style.height) || 0) / 2;
        if (Math.sqrt(Math.pow(px - 62.5, 2) + Math.pow(py - 46.4, 2)) > 4) {
            console.log("  => [智能初始化] 复位碾子以防遮挡...");
            await simulateDrag({page: 1, sliderId: 'page1panel20-271', startCoord: 'auto', endCoord: {x: 62.5, y: 46.4}, valueChange: {from: 0, to: 1}, wait: 1200, skipNav: true});
        }
    }

    // 器皿的目标安放中心位置
    const POT_CENTER = {x: 49, y: 38};

    const DOM_ORDER = [
        'page1panel20-547', // 蓝莓
        'page1panel20-908', // 树莓
        'page1panel20-602', // 蜜蜂
        'page1panel20-496', // 蚯蚓
        'page1panel20-432', // 蟋蟀
        'page1panel20-876', // 小卷
        'page1panel20-378', // 红伞
        'page1panel20-712', // 黄卷
        'page1panel20-948'  // 橘白
    ];

    // 4. 扫描所有 9 种材料，判断是否在目标位置，不在就移动
    for (let id of DOM_ORDER) {
        let el = document.getElementById(id);
        if (!el) continue;

        let currentX = (parseFloat(el.style.left) || 0) + (parseFloat(el.style.width) || 0) / 2;
        let currentY = (parseFloat(el.style.top) || 0) + (parseFloat(el.style.height) || 0) / 2;

        let isRequired = recipeIds.includes(id);
        let startX = INGREDIENTS[id].start.x;
        let startY = INGREDIENTS[id].start.y;

        let distToStart = Math.sqrt(Math.pow(currentX - startX, 2) + Math.pow(currentY - startY, 2));
        let distToPot = Math.sqrt(Math.pow(currentX - POT_CENTER.x, 2) + Math.pow(currentY - POT_CENTER.y, 2));

        let isAtStart = (distToStart <= 5);
        let isAtCenter = (distToPot <= 5);

        if (isRequired) {
            if (!isAtCenter) {
                console.log(`  => [配药] 将所需材料 [${INGREDIENTS[id].name}] 放入器皿中心...`);
                await simulateDrag({page: 1, sliderId: id, startCoord: 'auto', endCoord: POT_CENTER, valueChange: {from: 0, to: 1}, wait: 1200, skipNav: true});
            }
        } else {
            if (!isAtStart) {
                console.log(`  => [清理] 将多余材料 [${INGREDIENTS[id].name}] 移回初始位置...`);
                await simulateDrag({page: 1, sliderId: id, startCoord: 'auto', endCoord: INGREDIENTS[id].start, valueChange: {from: 0, to: 1}, wait: 1200, skipNav: true});
            }
        }
    }

    // 5. 碾子终极捣药
    console.log("  => 开始捣药！(碾子移动到指定坐标 x:54, y:28)");
    await simulateDrag({page: 1, sliderId: 'page1panel20-271', startCoord: 'auto', endCoord: {x: 54, y: 28}, valueChange: {from: 0, to: 1}, wait: 4000, skipNav: true});
}

// ==================== 完整操作表 ====================

window.MANUAL = [
    // ========== Manual 1: 窗户、沙漏 ==========
    {name: "窗户、沙漏", actions: [
        {type: 'click', page: 0, panel: 0, x: 36, y: 44, wait: 3000},
        {type: 'click', page: 0, panel: 1, x: 80, y: 65, wait: 5000}
    ]},

    // ========== Manual 2: 点击 Manifestum 得到新图片 ==========
    {name: "点击 Manifestum 得到新图片", actions: [
        {type: 'click', page: 0, panel: 14, x: 50, y: 49, wait: 800},
        {type: 'click', page: 0, panel: 2, x: 46, y: 48, wait: 800},
        {type: 'click', page: 0, panel: 15, x: 45, y: 34, wait: 800},
        {type: 'click', page: 0, panel: 10, x: 56, y: 32, wait: 800},
        {type: 'click', page: 0, panel: 7, x: 56, y: 33, wait: 800},
        {type: 'click', page: 0, panel: 6, x: 45, y: 44, wait: 800},
        {type: 'click', page: 0, panel: 20, x: 54, y: 48, wait: 800},
        {type: 'click', page: 0, panel: 21, x: 57, y: 46, wait: 800},
        {type: 'click', page: 0, panel: 22, x: 53, y: 46, wait: 800},
        {type: 'click', page: 0, panel: 14, x: 52, y: 63, wait: 4000}
    ]},

    // ========== Manual 3: 点击 Wagstaff 得到新图片 ==========
    {name: "点击 Wagstaff 得到新图片", actions: [
        {type: 'click', page: 0, panel: 24, x: 46, y: 45, wait: 800},
        {type: 'click', page: 0, panel: 2, x: 26, y: 57, wait: 800},
        {type: 'click', page: 0, panel: 8, x: 62, y: 46, wait: 800},
        {type: 'click', page: 0, panel: 20, x: 34, y: 47, wait: 800},
        {type: 'click', page: 0, panel: 21, x: 36, y: 47, wait: 800},
        {type: 'click', page: 0, panel: 2, x: 55, y: 50, wait: 800},
        {type: 'click', page: 0, panel: 7, x: 33, y: 40, wait: 800},
        {type: 'click', page: 0, panel: 7, x: 33, y: 40, wait: 4000}
    ]},

    // ========== Manual 4: 点击沙漏 ==========
    {name: "点击沙漏", actions: [
        {type: 'click', page: 0, panel: 1, x: 82, y: 61, wait: 3000}
    ]},

    // ========== Manual 5: 抽出纸条 ==========
    {name: "抽出纸条", actions: [
        {type: 'click', page: 0, panel: 9, x: 61, y: 66, wait: 2000}
    ]},

    // ========== Manual 6: 连接电线 ==========
    {name: "连接电线", actions: [
        {type: 'click', page: 0, panel: 4, x: 19, y: 48, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 75, y: 64, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 46, y: 60, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 46, y: 60, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 75, y: 67, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 17, y: 52, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 78, y: 65, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 46, y: 56, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 16, y: 48, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 16, y: 55, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 43, y: 60, wait: 4000}
    ]},

    // ========== Manual 7: 操作台 (操纵杆+滑块+按钮) ==========
    {name: "操作台 (操纵杆+滑块+按钮)", actions: [
        // 动态初始化：更换保险丝与开关判定
        async function() {
            if (getCurrentPage() !== '0') await navigateToPage('0');
            console.log("  => [初始化] 尝试更换保险丝...");
            await simulateClick({page: 0, panel: 9, x: 37, y: 90, wait: 1000, skipNav: true});
            await simulateClick({page: 0, panel: 9, x: 40, y: 88, wait: 1000, skipNav: true});
            await simulateClick({page: 0, panel: 9, x: 40, y: 88, wait: 1000, skipNav: true});

            console.log("  => [初始化] 检查开关状态...");
            let switch1 = document.getElementById('panel9switch1');
            if (switch1 && switch1.classList.contains('toggled')) {
                console.log("  => [初始化] 开关已开启(toggled)，执行点击...");
                await simulateClick({page: 0, panel: 9, x: 13, y: 60, wait: 1000, skipNav: true});
            }
        },
        // 初始点击启动操纵杆
        {type: 'click', page: 0, panel: 9, x: 9, y: 42, wait: 1000},
        // 第一组滑块操作
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 56, y: 83}, endCoord: {x: 54, y: 68}, valueChange: {from: 0, to: 15.2299}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 31, y: 84}, endCoord: {x: 38, y: 76}, valueChange: {from: 0, to: 10.6658}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 50, y: 69}, endCoord: {x: 54, y: 34}, valueChange: {from: 15.2299, to: 38.0747}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 63, y: 78}, endCoord: {x: 52, y: 54}, valueChange: {from: 0, to: 22.8448}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 41, y: 74}, endCoord: {x: 44, y: 66}, valueChange: {from: 10.6658, to: 21.3316}, wait: 1500},
        // 第一组按钮点击
        {type: 'click', page: 0, panel: 9, x: 59, y: 35, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 62, y: 37, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 58, y: 50, wait: 1000},
        // 第二组滑块操作
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 43, y: 47}, endCoord: {x: 45, y: 53}, valueChange: {from: 38.0747, to: 22.8448}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 41, y: 63}, endCoord: {x: 46, y: 55}, valueChange: {from: 21.3316, to: 31.9974}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 52, y: 58}, endCoord: {x: 63, y: 95}, valueChange: {from: 22.8448, to: 0}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 45, y: 60}, endCoord: {x: 65, y: 95}, valueChange: {from: 22.8448, to: 0}, wait: 1500},
        // 第二组按钮点击
        {type: 'click', page: 0, panel: 9, x: 65, y: 40, wait: 1000},
        // 第三组滑块操作
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 63, y: 81}, endCoord: {x: 61, y: 82}, valueChange: {from: 0, to: 7.61494}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 49, y: 55}, endCoord: {x: 59, y: 40}, valueChange: {from: 31.9974, to: 42.6632}, wait: 1500},
        // 第三组按钮点击
        {type: 'click', page: 0, panel: 9, x: 63, y: 52, wait: 1000},
        // 第四组滑块操作
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 59, y: 75}, endCoord: {x: 50, y: 32}, valueChange: {from: 7.61494, to: 38.0747}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 65, y: 81}, endCoord: {x: 54, y: 60}, valueChange: {from: 0, to: 22.8448}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 49, y: 45}, endCoord: {x: 55, y: 27}, valueChange: {from: 42.6632, to: 53.329}, wait: 1500},
        // 第五组滑块操作
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 47, y: 46}, endCoord: {x: 48, y: 49}, valueChange: {from: 38.0747, to: 30.4598}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 50, y: 60}, endCoord: {x: 52, y: 49}, valueChange: {from: 22.8448, to: 30.4598}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 51, y: 36}, endCoord: {x: 62, y: 12}, valueChange: {from: 53.329, to: 63.9948}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 52, y: 52}, endCoord: {x: 54, y: 65}, valueChange: {from: 30.4598, to: 15.2299}, wait: 1500},
        // 结束按钮点击组
        {type: 'click', page: 0, panel: 9, x: 35, y: 45, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 59, y: 50, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 80, y: 56, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 63, y: 51, wait: 1000},
        // 最后滑块操作
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 48, y: 68}, endCoord: {x: 48, y: 44}, valueChange: {from: 15.2299, to: 30.4598}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 50, y: 55}, endCoord: {x: 57, y: 60}, valueChange: {from: 30.4598, to: 22.8448}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 52, y: 53}, endCoord: {x: 45, y: 37}, valueChange: {from: 30.4598, to: 38.0747}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 56, y: 26}, endCoord: {x: 74, y: -12}, valueChange: {from: 63.9948, to: 74.6606}, wait: 5000}
    ]},

    // ========== Manual 8: 踢东西 (5次) ==========
    {name: "踢东西 (5次)", actions: [
        {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 1000}, {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 1000},
        {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 1000}, {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 1000},
        {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 4000}
    ]},

    // ========== Manual 9: 新图 (耳朵/纸条/瓦) ==========
    {name: "新图 (耳朵/纸条/瓦)", actions: [
        {type: 'click', page: 0, panel: 33, x: 22, y: 49, wait: 1000},
        {type: 'click', page: 0, panel: 34, x: 53, y: 72, wait: 1000},
        {type: 'click', page: 0, panel: 35, x: 56, y: 55, wait: 4000}
    ]},

    // ========== Manual 10: 点击螺丝 (4个) ==========
    {name: "点击螺丝 (4个)", actions: [
        {type: 'click', page: 0, panel: 38, x: 61, y: 94, wait: 800}, {type: 'click', page: 0, panel: 38, x: 82, y: 70, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 61, y: 74, wait: 800}, {type: 'click', page: 0, panel: 38, x: 82, y: 93, wait: 4000}
    ]},

    // ========== Manual 11: 点小按钮 (11个) ==========
    {name: "点小按钮 (11个)", actions: [
        {type: 'click', page: 0, panel: 38, x: 76, y: 78, wait: 800}, {type: 'click', page: 0, panel: 38, x: 72, y: 79, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 72, y: 75, wait: 800}, {type: 'click', page: 0, panel: 38, x: 69, y: 76, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 69, y: 83, wait: 800}, {type: 'click', page: 0, panel: 38, x: 76, y: 83, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 76, y: 91, wait: 800}, {type: 'click', page: 0, panel: 38, x: 73, y: 92, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 72, y: 88, wait: 800}, {type: 'click', page: 0, panel: 38, x: 68, y: 88, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 69, y: 92, wait: 4000}
    ]},

    // ========== Manual 12: 点物品 ==========
    {name: "点物品", actions: [
        {type: 'click', page: 1, panel: 0, x: 94, y: 83, wait: 1000}, {type: 'click', page: 1, panel: 1, x: 66, y: 36, wait: 1000},
        {type: 'click', page: 1, panel: 1, x: 62, y: 54, wait: 1000}, {type: 'click', page: 1, panel: 1, x: 78, y: 53, wait: 1000}
    ]},

    // ========== Manual 13: 树液桶 ==========
    {name: "树液桶", actions: [{type: 'click', page: 1, panel: 3, x: 40, y: 77, wait: 5000}]},

    // ========== Manual 14: 拿钥匙 ==========
    {name: "拿钥匙", actions: [
        async function() {
            if (getCurrentPage() !== '0') await navigateToPage('0');

            let drawer = document.getElementById('page0panel1drawer');
            if (!drawer) {
                console.log("  => 抽屉未打开，尝试点击打开...");
                await simulateClick({page: 0, panel: 1, x: 80, y: 91, wait: 2000, skipNav: true});
            }

            console.log("  => 拿取钥匙...");
            await simulateClick({page: 0, panel: '1drawer', x: 47, y: 40, wait: 1500, skipNav: true});

            console.log("  => 关闭抽屉...");
            await simulateClick({page: 0, panel: 1, x: 80, y: 91, wait: 1500, skipNav: true});
        }
    ]},

    // ========== Manual 15: 拿纸条 ==========
    {name: "拿纸条", actions: [
        async function() {
            if (getCurrentPage() !== '1') await navigateToPage('1');

            let toolbox = document.getElementById('page1panel4toolbox');
            if (!toolbox) {
                console.log("  => 工具箱未点开，尝试点击开启...");
                await simulateClick({page: 1, panel: 4, x: 7, y: 35, wait: 2000, skipNav: true});
            }

            // 1. 检查锁是否已经打开过（看内部的三张纸条是否存在）
            const notes = ['page1panel4toolboxhatchnote', 'page1panel4toolboxbushnote', 'page1panel4toolboxdepositslip'];
            let hasNotes = notes.some(id => document.getElementById(id) !== null);

            if (hasNotes) {
                console.log("  => [智能判断] 发现锁内纸条，说明钥匙已开！直接拖出纸条...");
                for (let noteId of notes) {
                    if (document.getElementById(noteId)) {
                        console.log(`  => 📄 将纸条 ${noteId} 拖出...`);
                        await simulateDrag({ page: 1, sliderId: noteId, startCoord: 'auto', endCoord: {x: 94, y: 40}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true });
                    }
                }
                console.log("  => ✅ 操作结束，直接关闭工具箱。");
                await simulateClick({page: 1, panel: 4, x: 7, y: 35, wait: 1000, skipNav: true});
                return; // 提前结束
            }

            // 2. 如果没发现纸条，检查工具箱面板和钥匙状态
            let panel = document.getElementById('page1panel4toolboxpanel');
            if (panel) {
                let key = document.getElementById('page1panel4toolboxkey');
                if (!key) {
                    console.log("  => ⚠ 发现钥匙缺失！自动执行补救措施: 回去拿钥匙...");
                    await navigateToPage('0');

                    let drawer = document.getElementById('page0panel1drawer');
                    if (!drawer) {
                        await simulateClick({page: 0, panel: 1, x: 80, y: 91, wait: 2000, skipNav: true});
                    }
                    await simulateClick({page: 0, panel: '1drawer', x: 47, y: 40, wait: 1500, skipNav: true});
                    await simulateClick({page: 0, panel: 1, x: 80, y: 91, wait: 1500, skipNav: true});

                    await navigateToPage('1');
                    await simulateClick({page: 1, panel: 4, x: 7, y: 35, wait: 2000, skipNav: true});
                }

                key = document.getElementById('page1panel4toolboxkey');
                if (key) {
                    let hammer = document.getElementById('page1panel4toolboxhammer');
                    if (hammer) {
                        console.log("  => 移开锤子...");
                        await simulateDrag({ page: 1, sliderId: 'page1panel4toolboxhammer', startCoord: 'auto', endCoord: {x: 62, y: 54}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true });
                    }
                    console.log("  => 移开木板...");
                    await simulateDrag({ page: 1, sliderId: 'page1panel4toolboxpanel', startCoord: 'auto', endCoord: {x: 13, y: 73}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true });

                    console.log("  => 🔑 识别到钥匙位置，进行拖动开锁...");
                    await simulateDrag({ page: 1, sliderId: 'page1panel4toolboxkey', startCoord: 'auto', endCoord: {x: 33, y: 74}, valueChange: {from: 0, to: 1}, wait: 2000, skipNav: true });

                    for (let noteId of notes) {
                        if (document.getElementById(noteId)) {
                            console.log(`  => 📄 将纸条 ${noteId} 拖出...`);
                            await simulateDrag({ page: 1, sliderId: noteId, startCoord: 'auto', endCoord: {x: 94, y: 40}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true });
                        }
                    }

                    console.log("  => ✅ 操作结束，直接关闭工具箱。");
                    await simulateClick({page: 1, panel: 4, x: 7, y: 35, wait: 1000, skipNav: true});
                }
            } else {
                console.log("  => ✅ 工具箱内已清空，直接关闭工具箱。");
                await simulateClick({page: 1, panel: 4, x: 7, y: 35, wait: 1000, skipNav: true});
            }
        }
    ]},

    // ========== Manual 16: 浆果丛 ==========
    {name: "浆果丛", actions: [
        {type: 'click', page: 1, panel: 11, x: 69, y: 57, wait: 1000}, {type: 'click', page: 1, panel: 11, x: 77, y: 76, wait: 1000},
        {type: 'click', page: 1, panel: 11, x: 94, y: 76, wait: 1000}, {type: 'click', page: 1, panel: 11, x: 86, y: 56, wait: 1000},
        {type: 'click', page: 1, panel: 11, x: 55, y: 76, wait: 1000}
    ]},

    // ========== Manual 17: 点灌木丛找蟋蟀 ==========
    {name: "点灌木丛找蟋蟀", actions: [
        async function() {
            let panel6 = document.getElementById('page1panel6');
            if (!panel6) {
                console.log("  => 正在点击灌木丛，请等待虫子出现...");
                await simulateClick({page: 1, panel: 7, x: 9, y: 70, wait: 5000, skipNav: true});
            }
        }
    ]},

    // ========== Manual 18: 蟋蟀和蘑菇 ==========
    {name: "蟋蟀和蘑菇", actions: [
        async function() {
            let panel6 = document.getElementById('page1panel6');
            if (!panel6) {
                console.log("  => [辅助采集] 虫子没出来，尝试点击灌木丛...");
                await simulateClick({page: 1, panel: 7, x: 9, y: 70, wait: 2000, skipNav: true});
                await waitForElement('#page1panel6');
            }
            await simulateClick({page: 1, panel: 6, x: 26, y: 77, wait: 800, skipNav: true});
            await simulateClick({page: 1, panel: 6, x: 83, y: 84, wait: 800, skipNav: true});
        }
    ]},

    // ========== Manual 19: 收集各种材料 ==========
    {name: "收集各种材料", actions: [
        {type: 'click', page: 1, panel: 4, x: 51, y: 52, wait: 1000}, {type: 'click', page: 1, panel: 5, x: 7, y: 16, wait: 1000},
        {type: 'click', page: 1, panel: 5, x: 37, y: 89, wait: 1000}, {type: 'click', page: 1, panel: 7, x: 20, y: 80, wait: 1000},
        {type: 'click', page: 1, panel: 7, x: 37, y: 70, wait: 1000}, {type: 'click', page: 1, panel: 7, x: 80, y: 81, wait: 1000},
        {type: 'click', page: 1, panel: 8, x: 13, y: 82, wait: 1000}, {type: 'click', page: 1, panel: 8, x: 36, y: 82, wait: 1000},
        {type: 'click', page: 1, panel: 10, x: 32, y: 85, wait: 1000}, {type: 'click', page: 1, panel: 10, x: 65, y: 63, wait: 1000},
        {type: 'click', page: 1, panel: 12, x: 92, y: 12, wait: 4000}
    ]},

    // ========== Manual 20: 打开井盖并入井 ==========
    {name: "打开井盖并入井", actions: [
        {type: 'click', page: 1, panel: 12, x: 63, y: 70, wait: 800}, {type: 'click', page: 1, panel: 12, x: 63, y: 70, wait: 800},
        {type: 'click', page: 1, panel: 12, x: 58, y: 81, wait: 800}, {type: 'click', page: 1, panel: 12, x: 63, y: 70, wait: 800},
        {type: 'click', page: 1, panel: 12, x: 70, y: 81, wait: 800}, {type: 'click', page: 1, panel: 12, x: 58, y: 78, wait: 800},
        {type: 'click', page: 1, panel: 12, x: 58, y: 78, wait: 800}, {type: 'click', page: 1, panel: 12, x: 63, y: 70, wait: 800},
        {type: 'click', page: 1, panel: 12, x: 71, y: 79, wait: 4000},
        {type: 'click', page: 1, panel: 13, x: 42, y: 70, wait: 2000}
    ]},

    // ========== Manual 21: 药水配方一 ==========
    {name: "药水配方一", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-948', 'page1panel20-908', 'page1panel20-712', 'page1panel20-432']);
        }
    ]},

    // ========== Manual 22: 药水配方二 ==========
    {name: "药水配方二", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-378', 'page1panel20-908', 'page1panel20-712', 'page1panel20-432']);
        }
    ]},

    // ========== Manual 23: 药水配方三 ==========
    {name: "药水配方三", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-876', 'page1panel20-908', 'page1panel20-712', 'page1panel20-432']);
        }
    ]},

    // ========== Manual 24: 药水配方四 ==========
    {name: "药水配方四", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-876', 'page1panel20-547', 'page1panel20-712', 'page1panel20-602', 'page1panel20-948']);
        }
    ]},

    // ========== Manual 25: 药水配方五 ==========
    {name: "药水配方五", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-908', 'page1panel20-432', 'page1panel20-547', 'page1panel20-496']);
        }
    ]},

    // ========== Manual 26: 打开音乐之门 ==========
    {name: "打开音乐之门", actions: [
        async function() {
            if (getCurrentPage() !== '1') await navigateToPage('1');

            // 检查音乐之门是否已经打开 (存在门上的小纸条)
            let screwdriver = document.getElementById('page1panel16screwdriver');
            if (screwdriver) {
                console.log("  => [智能判断] 发现小纸条，音乐之门已打开，跳过解谜步骤。");
                return;
            }

            console.log("  => 点击音乐钟激活门...");
            await simulateClick({page: 1, panel: 16, x: 17, y: 50, wait: 2000, skipNav: true});

            console.log("  => 开始按顺序点击音乐之门...");
            const musicSteps = [
                {x: 54, y: 74}, {x: 91, y: 21}, {x: 68, y: 61}, {x: 67, y: 90},
                {x: 92, y: 47}, {x: 68, y: 61}, {x: 92, y: 19}, {x: 66, y: 42},
                {x: 67, y: 90}, {x: 92, y: 47}, {x: 59, y: 35}, {x: 92, y: 19},
                {x: 39, y: 35}, {x: 68, y: 91}, {x: 92, y: 47}, {x: 58, y: 35},
                {x: 92, y: 20}, {x: 39, y: 35}, {x: 67, y: 91}, {x: 92, y: 46},
                {x: 44, y: 74}, {x: 68, y: 61}, {x: 58, y: 35}, {x: 68, y: 90}
            ];
            for (let step of musicSteps) {
                await simulateClick({page: 1, panel: 17, x: step.x, y: step.y, wait: 1500, skipNav: true});
            }
        }
    ]},

    // ========== Manual 27: 点击 Vestigo 出现土坑 ==========
    {name: "点击 Vestigo 出现土坑", actions: [
        {type: 'click', page: 1, panel: 0, x: 56, y: 77, wait: 1000}, {type: 'click', page: 1, panel: 0, x: 35, y: 57, wait: 1000},
        {type: 'click', page: 1, panel: 0, x: 45, y: 76, wait: 1000}, {type: 'click', page: 1, panel: 0, x: 43, y: 56, wait: 1000},
        {type: 'click', page: 1, panel: 0, x: 64, y: 77, wait: 1000}, {type: 'click', page: 1, panel: 0, x: 25, y: 77, wait: 1000},
        {type: 'click', page: 1, panel: 0, x: 27, y: 58, wait: 4000}
    ]},

    // ========== Manual 28: 护目镜调焦 ==========
    {name: "护目镜调焦", actions: [
        {type: 'click', page: 4, panel: 0, x: 26, y: 28, wait: 5000},
        {type: 'drag', page: 4, sliderId: 'page4panel1dial1', startCoord: {x: 27, y: 9}, endCoord: {x: 24, y: 10}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page4panel1dial2', startCoord: {x: 36, y: 76}, endCoord: {x: 31, y: 79}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page4panel1dial3', startCoord: {x: 71, y: 10}, endCoord: {x: 78, y: 11}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page4panel1dial4', startCoord: {x: 70, y: 83}, endCoord: {x: 78, y: 81}, valueChange: {from: 0, to: 1}, wait: 3000}
    ]},

    // ========== Manual 29: 寻找零部件（5个） ==========
    {name: "寻找零部件（5个）", actions: [
        {type: 'click', page: 4, panel: 2, x: 16, y: 18, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 33, y: 66, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 71, y: 47, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 40, y: 75, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 86, y: 47, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 33, y: 34, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 30, y: 75, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 24, y: 48, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 27, y: 41, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 40, y: 82, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 63, y: 48, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 42, y: 63, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 50, y: 96, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 20, y: 51, wait: 1000}
    ]},

    // ========== Manual 30: 获得留声机 ==========
    {name: "获得留声机", actions: [
        {type: 'click', page: 2, panel: 0, x: 5, y: 49, wait: 3000},
        {type: 'click', page: 2, panel: 1, x: 78, y: 43, wait: 3000}
    ]},

    // ========== Manual 31: 调整激光透镜 ==========
    {name: "调整激光透镜", actions: [
        // 动态初始化：智能打开滤镜窗并消除干扰光线，最后将操作条向下移到底部
        async function() {
            if (getCurrentPage() !== '5') await navigateToPage('5');

            let panel1 = document.getElementById('page5panel1');
            if (!panel1) {
                console.log("  => [初始化] 滤镜窗未打开，尝试点击打开...");
                await simulateClick({page: 5, panel: 0, x: 82, y: 61, wait: 4000, skipNav: true});
            }

            // 仅在初始化阶段判断未关闭的盒子
            if (document.getElementById('page5panel1openbox')) {
                console.log("  => [初始化] 发现未关闭的盒子，执行一次复位...");
                await simulateClick({page: 5, panel: 1, x: 0, y: 0, wait: 1000, skipNav: true, skipCheck: true});
            }

            console.log("  => [初始化] 1. 取消所有滤镜...");
            await simulateClick({page: 5, panel: 1, x: 11, y: 73, wait: 1500, skipNav: true});

            console.log("  => [初始化] 2. 检查滑块位置...");
            let indicator = document.getElementById('page5panel1slotindicator');
            let maxMoves = 30;
            while (indicator && !indicator.style.top.includes('77') && maxMoves > 0) {
                console.log(`  => 当前滑块位置: ${indicator.style.top}, 继续向下移动...`);
                await simulateClick({page: 5, panel: 1, x: 91, y: 90, wait: 1000, skipNav: true});
                maxMoves--;
            }
            if (maxMoves === 0) {
                console.warn("  => [警告] 滑块移动似乎达到上限或未找到目标位置，继续执行。");
            }

            console.log("  => [初始化] 3. 检查光束是否存在...");
            const beams = [
                { id: 'page5panel1lbeam', x: 36, y: 94 },
                { id: 'page5panel1mbeam', x: 46, y: 94 },
                { id: 'page5panel1hbeam', x: 56, y: 94 },
                { id: 'page5panel1abeam', x: 66, y: 94 }
            ];
            for (let beam of beams) {
                if (document.getElementById(beam.id)) {
                    console.log(`  => 发现存在 ${beam.id}，执行清除...`);
                    await simulateClick({page: 5, panel: 1, x: beam.x, y: beam.y, wait: 1500, skipNav: true});
                }
            }
            console.log("  => [初始化完成] 开始执行主透镜调整序列。");
        },

        ...(function() {
            const WAIT_TIME = 1000;
            const steps = [
                {type: 'click', page: 5, panel: 1, x: 35, y: 93}, {type: 'click', page: 5, panel: 1, x: 92, y: 54},
                {type: 'click', page: 5, panel: 1, x: 49, y: 42}, {type: 'click', page: 5, panel: 1, x: 16, y: 88},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 92, y: 54},
                {type: 'click', page: 5, panel: 1, x: 48, y: 17}, {type: 'click', page: 5, panel: 1, x: 35, y: 93},
                {type: 'click', page: 5, panel: 1, x: 45, y: 93}, {type: 'click', page: 5, panel: 1, x: 55, y: 93},
                {type: 'click', page: 5, panel: 1, x: 6, y: 86}, {type: 'click', page: 5, panel: 1, x: 91, y: 55},
                {type: 'click', page: 5, panel: 1, x: 49, y: 42}, {type: 'click', page: 5, panel: 1, x: 15, y: 89},
                {type: 'click', page: 5, panel: 1, x: 79, y: 90}, {type: 'click', page: 5, panel: 1, x: 92, y: 54},
                {type: 'click', page: 5, panel: 1, x: 48, y: 27}, {type: 'click', page: 5, panel: 1, x: 15, y: 89},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 92, y: 54},
                {type: 'click', page: 5, panel: 1, x: 48, y: 27}, {type: 'click', page: 5, panel: 1, x: 35, y: 94},
                {type: 'click', page: 5, panel: 1, x: 11, y: 73}, {type: 'click', page: 5, panel: 1, x: 91, y: 90},
                {type: 'click', page: 5, panel: 1, x: 91, y: 90}, {type: 'click', page: 5, panel: 1, x: 91, y: 90},
                {type: 'click', page: 5, panel: 1, x: 92, y: 55}, {type: 'click', page: 5, panel: 1, x: 48, y: 78},
                {type: 'click', page: 5, panel: 1, x: 14, y: 89}, {type: 'click', page: 5, panel: 1, x: 78, y: 89},
                {type: 'click', page: 5, panel: 1, x: 92, y: 54}, {type: 'click', page: 5, panel: 1, x: 49, y: 57},
                {type: 'click', page: 5, panel: 1, x: 15, y: 89}, {type: 'click', page: 5, panel: 1, x: 79, y: 89},
                {type: 'click', page: 5, panel: 1, x: 92, y: 55}, {type: 'click', page: 5, panel: 1, x: 49, y: 57},
                {type: 'click', page: 5, panel: 1, x: 14, y: 90}, {type: 'click', page: 5, panel: 1, x: 79, y: 89},
                {type: 'click', page: 5, panel: 1, x: 92, y: 54}, {type: 'click', page: 5, panel: 1, x: 48, y: 57},
                {type: 'click', page: 5, panel: 1, x: 11, y: 73}, {type: 'click', page: 5, panel: 1, x: 91, y: 90},
                {type: 'click', page: 5, panel: 1, x: 91, y: 90}, {type: 'click', page: 5, panel: 1, x: 91, y: 90},
                {type: 'click', page: 5, panel: 1, x: 35, y: 93}, {type: 'click', page: 5, panel: 1, x: 45, y: 94},
                {type: 'click', page: 5, panel: 1, x: 92, y: 54}, {type: 'click', page: 5, panel: 1, x: 49, y: 47},
                {type: 'click', page: 5, panel: 1, x: 11, y: 73}, {type: 'click', page: 5, panel: 1, x: 35, y: 94},
                {type: 'click', page: 5, panel: 1, x: 46, y: 93}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                {type: 'click', page: 5, panel: 1, x: 81, y: 22}, {type: 'click', page: 5, panel: 1, x: 14, y: 90},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                {type: 'click', page: 5, panel: 1, x: 80, y: 22}, {type: 'click', page: 5, panel: 1, x: 15, y: 90},
                ...Array.from({ length: 13 }, () => ({type: 'click', page: 5, panel: 1, x: 79, y: 89})),
                {type: 'click', page: 5, panel: 1, x: 91, y: 54}, {type: 'click', page: 5, panel: 1, x: 80, y: 78},
                {type: 'click', page: 5, panel: 1, x: 11, y: 73},
                ...Array.from({ length: 9 }, () => ({type: 'click', page: 5, panel: 1, x: 90, y: 89})),
                {type: 'click', page: 5, panel: 1, x: 91, y: 54}, {type: 'click', page: 5, panel: 1, x: 49, y: 18},
                {type: 'click', page: 5, panel: 1, x: 14, y: 90}, {type: 'click', page: 5, panel: 1, x: 91, y: 89},
                {type: 'click', page: 5, panel: 1, x: 92, y: 54}, {type: 'click', page: 5, panel: 1, x: 47, y: 42},
                {type: 'click', page: 5, panel: 1, x: 14, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 89},
                {type: 'click', page: 5, panel: 1, x: 92, y: 54}, {type: 'click', page: 5, panel: 1, x: 48, y: 42},
                {type: 'click', page: 5, panel: 1, x: 11, y: 73}, {type: 'click', page: 5, panel: 1, x: 66, y: 93},
                {type: 'click', page: 5, panel: 1, x: 91, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 89},
                {type: 'click', page: 5, panel: 1, x: 91, y: 54}, {type: 'click', page: 5, panel: 1, x: 80, y: 17},
                {type: 'click', page: 5, panel: 1, x: 14, y: 90}, {type: 'click', page: 5, panel: 1, x: 91, y: 89},
                {type: 'click', page: 5, panel: 1, x: 92, y: 54}, {type: 'click', page: 5, panel: 1, x: 49, y: 17},
                {type: 'click', page: 5, panel: 1, x: 14, y: 90},
                ...Array.from({ length: 3 }, () => ({type: 'click', page: 5, panel: 1, x: 79, y: 89})),
                ...Array.from({ length: 3 }).flatMap(() => [
                    {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                    {type: 'click', page: 5, panel: 1, x: 48, y: 42}, {type: 'click', page: 5, panel: 1, x: 14, y: 90}
                ]),
                {type: 'click', page: 5, panel: 1, x: 11, y: 73}, {type: 'click', page: 5, panel: 1, x: 91, y: 89},
                {type: 'click', page: 5, panel: 1, x: 91, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 89},
                {type: 'click', page: 5, panel: 1, x: 91, y: 54}, {type: 'click', page: 5, panel: 1, x: 81, y: 44},
                {type: 'click', page: 5, panel: 1, x: 14, y: 90}, {type: 'click', page: 5, panel: 1, x: 90, y: 89},
                {type: 'click', page: 5, panel: 1, x: 90, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                {type: 'click', page: 5, panel: 1, x: 80, y: 22}, {type: 'click', page: 5, panel: 1, x: 14, y: 89},
                {type: 'click', page: 5, panel: 1, x: 91, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                {type: 'click', page: 5, panel: 1, x: 81, y: 45}, {type: 'click', page: 5, panel: 1, x: 14, y: 89},
                {type: 'click', page: 5, panel: 1, x: 79, y: 90}, {type: 'click', page: 5, panel: 1, x: 79, y: 90},
                {type: 'click', page: 5, panel: 1, x: 79, y: 90}, {type: 'click', page: 5, panel: 1, x: 5, y: 86},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 92, y: 54},
                {type: 'click', page: 5, panel: 1, x: 81, y: 22}, {type: 'click', page: 5, panel: 1, x: 14, y: 90},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                {type: 'click', page: 5, panel: 1, x: 48, y: 22}, {type: 'click', page: 5, panel: 1, x: 11, y: 73},
                {type: 'click', page: 5, panel: 1, x: 91, y: 90}, {type: 'click', page: 5, panel: 1, x: 91, y: 90},
                {type: 'click', page: 5, panel: 1, x: 91, y: 90}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                {type: 'click', page: 5, panel: 1, x: 48, y: 57}, {type: 'click', page: 5, panel: 1, x: 14, y: 89},
                {type: 'click', page: 5, panel: 1, x: 91, y: 90}, {type: 'click', page: 5, panel: 1, x: 92, y: 54},
                {type: 'click', page: 5, panel: 1, x: 48, y: 57}, {type: 'click', page: 5, panel: 1, x: 66, y: 93},
                {type: 'click', page: 5, panel: 1, x: 66, y: 93}, {type: 'click', page: 5, panel: 1, x: 14, y: 90},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 79, y: 89},
                {type: 'click', page: 5, panel: 1, x: 92, y: 54}, {type: 'click', page: 5, panel: 1, x: 48, y: 42},
                {type: 'click', page: 5, panel: 1, x: 5, y: 86}, {type: 'click', page: 5, panel: 1, x: 35, y: 94},
                {type: 'click', page: 5, panel: 1, x: 66, y: 94}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                {type: 'click', page: 5, panel: 1, x: 48, y: 57}, {type: 'click', page: 5, panel: 1, x: 14, y: 90},
                {type: 'click', page: 5, panel: 1, x: 91, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 89},
                {type: 'click', page: 5, panel: 1, x: 91, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                {type: 'click', page: 5, panel: 1, x: 48, y: 78}, {type: 'click', page: 5, panel: 1, x: 35, y: 93},
                {type: 'click', page: 5, panel: 1, x: 11, y: 73}, {type: 'click', page: 5, panel: 1, x: 66, y: 93},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 92, y: 54},
                {type: 'click', page: 5, panel: 1, x: 80, y: 22}, {type: 'click', page: 5, panel: 1, x: 14, y: 90},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 91, y: 54},
                {type: 'click', page: 5, panel: 1, x: 82, y: 31}, {type: 'click', page: 5, panel: 1, x: 15, y: 89},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 92, y: 54},
                {type: 'click', page: 5, panel: 1, x: 81, y: 45}, {type: 'click', page: 5, panel: 1, x: 15, y: 89},
                {type: 'click', page: 5, panel: 1, x: 79, y: 89}, {type: 'click', page: 5, panel: 1, x: 79, y: 89},
                {type: 'click', page: 5, panel: 1, x: 91, y: 54}, {type: 'click', page: 5, panel: 1, x: 48, y: 57},
                {type: 'click', page: 5, panel: 1, x: 14, y: 90}, {type: 'click', page: 5, panel: 1, x: 79, y: 89},
                {type: 'click', page: 5, panel: 1, x: 91, y: 54}, {type: 'click', page: 5, panel: 1, x: 49, y: 57}
            ];
            return steps.map(action => ({ ...action, wait: WAIT_TIME }));
        })()
    ]},

    // ========== Manual 32: 获取透镜 ==========
    {name: "获取透镜", actions: [
        {type: 'drag', page: 4, sliderId: 'page4panel1dial5', startCoord: {x: 9, y: 60}, endCoord: {x: 12, y: 70}, valueChange: {from: 0, to: 1}, wait: 3000}
    ]},

    // ========== Manual 33: 寻找花瓣（17片） ==========
    {name: "寻找花瓣（17片）", actions: [
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: 'auto', endCoord: {x: 87.5, y: 69.7}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 2, x: 56, y: 61, wait: 1500}, {type: 'click', page: 4, panel: 3, x: 40, y: 82, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 87, y: 69}, endCoord: {x: 87, y: 56}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 85, y: 54, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 77, y: 45, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 32, y: 85, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 89, y: 54}, endCoord: {x: 37, y: 46}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 37, y: 46, wait: 1500}, {type: 'click', page: 4, panel: 3, x: 48, y: 70, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 37, y: 49, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 87, y: 47, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 42, y: 95, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 36, y: 48}, endCoord: {x: 20, y: 48}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 16, y: 44, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 21, y: 42, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 21, y: 48}, endCoord: {x: 74, y: 46}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 73, y: 44, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 78, y: 32, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 48, y: 79, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 75, y: 47}, endCoord: {x: 82, y: 56}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 80, y: 55, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 19, y: 30, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 33, y: 66, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 83, y: 54}, endCoord: {x: 60, y: 46}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 58, y: 45, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 71, y: 15, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 27, y: 72, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 59, y: 47}, endCoord: {x: 77, y: 47}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 75, y: 48, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 38, y: 58, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 50, y: 89, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 76, y: 47}, endCoord: {x: 45, y: 55}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 45, y: 54, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 37, y: 31, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 45, y: 96, wait: 1500}, {type: 'click', page: 4, panel: 3, x: 42, y: 50, wait: 1500},
        {type: 'click', page: 4, panel: 2, x: 82, y: 38, wait: 1500}, {type: 'click', page: 4, panel: 3, x: 43, y: 79, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 45, y: 54}, endCoord: {x: 54, y: 57}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 54, y: 57, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 78, y: 63, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 29, y: 85, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 54, y: 56}, endCoord: {x: 21, y: 48}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 21, y: 48, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 71, y: 51, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 27, y: 91, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 21, y: 50}, endCoord: {x: 64, y: 57}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 62, y: 55, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 19, y: 77, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 45, y: 76, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 64, y: 54}, endCoord: {x: 18, y: 48}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 18, y: 48, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 81, y: 11, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 32, y: 92, wait: 1500}, {type: 'click', page: 4, panel: 3, x: 16, y: 46, wait: 1500},
        {type: 'click', page: 4, panel: 2, x: 65, y: 85, wait: 1500}, {type: 'click', page: 4, panel: 3, x: 50, y: 96, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 17, y: 48}, endCoord: {x: 76, y: 53}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 76, y: 53, wait: 1500}, {type: 'click', page: 4, panel: 2, x: 89, y: 55, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 30, y: 66, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: {x: 77, y: 52}, endCoord: {x: 36, y: 47}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 3, x: 36, y: 47, wait: 1500}
    ]},

    // ========== Manual 34: 寻找其余零件（35个） ==========
    {name: "寻找其余零件（35个）", actions: [
        {type: 'drag', page: 4, sliderId: 'page5panel3monocle', startCoord: 'auto', endCoord: {x: 87.5, y: 69.7}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'click', page: 4, panel: 2, x: 76, y: 71, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 29, y: 85, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 63, y: 40, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 63, y: 52, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 40, y: 66, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 87, y: 44, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 27, y: 77, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 45, y: 79, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 90, y: 43, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 42, y: 64, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 40, y: 82, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 69, y: 53, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 57, y: 47, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 75, y: 51, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 57, y: 80, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 29, y: 92, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 92, y: 45, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 41, y: 85, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 27, y: 89, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 20, y: 52, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 53, y: 47, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 15, y: 48, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 35, y: 82, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 43, y: 83, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 64, y: 55, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 74, y: 78, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 46, y: 70, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 8, y: 47, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 88, y: 71, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 51, y: 76, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 70, y: 45, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 66, y: 70, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 45, y: 83, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 11, y: 47, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 42, y: 96, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 76, y: 49, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 66, y: 16, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 41, y: 54, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 77, y: 51, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 50, y: 80, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 25, y: 50, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 52, y: 68, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 47, y: 96, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 49, y: 51, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 52, y: 17, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 86, y: 50, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 28, y: 68, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 40, y: 89, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 69, y: 46, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 48, y: 63, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 32, y: 72, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 16, y: 51, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 61, y: 43, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 74, y: 48, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 83, y: 59, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 38, y: 69, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 87, y: 53, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 73, y: 57, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 37, y: 83, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 16, y: 50, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 38, y: 61, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 37, y: 95, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 67, y: 48, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 30, y: 58, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 43, y: 76, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 70, y: 57, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 16, y: 54, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 35, y: 76, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 8, y: 47, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 26, y: 50, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 27, y: 65, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 61, y: 44, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 49, y: 42, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 35, y: 79, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 28, y: 49, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 84, y: 32, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 45, y: 76, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 67, y: 46, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 70, y: 33, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 48, y: 83, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 28, y: 49, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 47, y: 33, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 51, y: 73, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 42, y: 49, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 36, y: 22, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 80, y: 50, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 32, y: 85, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 10, y: 50, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 27, y: 35, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 40, y: 92, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 59, y: 49, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 82, y: 23, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 29, y: 88, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 40, y: 50, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 58, y: 26, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 48, y: 86, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 27, y: 46, wait: 1000}
    ]},

    // ========== Manual 35: 音叉解密 ==========
    {name: "音叉解密", actions: [
        {type: 'click', page: 0, panel: 27, x: 22, y: 33, wait: 1000}, {type: 'click', page: 0, panel: 27, x: 29, y: 34, wait: 1000},
        {type: 'click', page: 0, panel: 27, x: 29, y: 34, wait: 1000}, {type: 'click', page: 0, panel: 27, x: 22, y: 33, wait: 1000},
        {type: 'click', page: 0, panel: 27, x: 22, y: 33, wait: 1000}, {type: 'click', page: 0, panel: 27, x: 29, y: 34, wait: 1000},
        {type: 'click', page: 0, panel: 27, x: 22, y: 33, wait: 1000}, {type: 'click', page: 0, panel: 27, x: 22, y: 33, wait: 1000},
        {type: 'click', page: 0, panel: 27, x: 22, y: 33, wait: 1000}, {type: 'click', page: 0, panel: 27, x: 29, y: 34, wait: 1000},
        {type: 'click', page: 0, panel: 27, x: 29, y: 34, wait: 1000}, {type: 'click', page: 0, panel: 27, x: 22, y: 33, wait: 1000},
        {type: 'click', page: 0, panel: 27, x: 22, y: 33, wait: 1000}, {type: 'click', page: 0, panel: 27, x: 29, y: 34, wait: 1000},
        {type: 'click', page: 0, panel: 27, x: 29, y: 34, wait: 1000}, {type: 'click', page: 0, panel: 27, x: 29, y: 34, wait: 1000},
        {type: 'click', page: 0, panel: 27, x: 22, y: 33, wait: 1000}
    ]}
];

// ==================== 调度控制器 ====================

const Controller = {
    config: window.MANUAL,
    currentStage: 0,
    isRunning: false,
    autoMode: false,

    list() {
        if (!this.config) return console.log('❌ 未找到配置');
        console.log(`\n${'━'.repeat(50)}\n📋 操作步骤列表：\n${'━'.repeat(50)}`);
        this.config.forEach((stage, idx) => {
            console.log(`${idx + 1}. ${stage.name}`);
        });
        console.log(`${'━'.repeat(50)}\n💡 输入 jumpTo(n) 空降到第 n 阶段\n`);
    },

    async executeStage(index) {
        if (!this.config || index < 0 || index >= this.config.length) return false;

        this.isRunning = true;
        const stage = this.config[index];

        console.log(`\n${'='.repeat(50)}\n正在执行阶段 ${index + 1}/${this.config.length}: ${stage.name}\n${'='.repeat(50)}`);

        try {
            for (let i = 0; i < stage.actions.length; i++) {
                if (!this.isRunning) return false;
                console.log(`[步骤 ${i + 1}/${stage.actions.length}]`);

                const action = stage.actions[i];

                if (typeof action === 'function') {
                    await action();
                } else if (action.type === 'click') {
                    await simulateClick(action);
                } else if (action.type === 'drag') {
                    await simulateDrag(action);
                }
            }
            console.log(`✓ 阶段 ${index + 1} 完成`);
            this.isRunning = false;
            return true;
        } catch (error) {
            if (error.message && error.message.includes("元素未找到")) {
                console.error(`❌ 执行中止: 未找到页面元素。\n\n💡 提示: ${error.message}。\n请确保前置解谜步骤已经做完，或者网络加载完成后再试，您可以尝试跳过(jumpTo)当前步骤。`);
            } else {
                console.error(`❌ 执行出错:`, error);
            }
            this.isRunning = false;
            return false;
        }
    },

    async next() {
        if (this.isRunning) return console.warn('⚠️ 正在执行中，请耐心等待完成');
        if (this.currentStage >= this.config.length) return console.log('\n🎉 所有阶段已完成！');

        const success = await this.executeStage(this.currentStage);
        if (success) {
            this.currentStage++;
            if (this.currentStage < this.config.length) {
                console.log(`💡 输入 next() 继续下一阶段`);
                if (this.autoMode) {
                    await sleep(1000);
                    await this.next();
                }
            } else {
                console.log('\n🎊 所有剧情解锁完毕，解谜圆满结束！');
            }
        }
    },

    async auto() {
        console.log(`\n🚀 启动全自动驾驶模式！共 ${this.config.length} 个阶段。\n`);
        this.autoMode = true;
        await this.next();
    },

    jumpTo(stageNumber) {
        const index = stageNumber - 1;
        if (index < 0 || index >= this.config.length) return console.error(`❌ 编号无效，请输入 1-${this.config.length}`);

        this.currentStage = index;
        console.log(`\n${'━'.repeat(50)}\n📍 已跳转到阶段 ${stageNumber}/${this.config.length} => [${this.config[index].name}]\n💡 输入 next() 开始执行。\n(⚠️ 脚本会在执行时自动导航到对应的页面)\n${'━'.repeat(50)}`);
    },

    stop() {
        this.isRunning = false;
        this.autoMode = false;
        console.log('🛑 已紧急停止执行');
    }
};

window.list = () => Controller.list();
window.next = () => Controller.next();
window.auto = () => Controller.auto();
window.jumpTo = (n) => Controller.jumpTo(n);
window.stop = () => Controller.stop();

console.log(`
${'━'.repeat(60)}
🎮 饥荒解谜自动化脚本 - 终极整合版 🎮
${'━'.repeat(60)}
✅ 已加载自动寻路系统
✅ 包含特判未关闭物品等智能保护逻辑
✅ 已合并共 ${window.MANUAL.length} 个解谜阶段

[命令清单]
  auto()      - 全自动通关
  next()      - 走一步看一步
  list()      - 查看所有操作步骤列表
  jumpTo(n)   - 空降到第 n 阶段 (附带自动跨图寻路)
  stop()      - 刹车
${'━'.repeat(60)}
`);

})();
```