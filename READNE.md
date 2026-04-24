# Usage
### Open the official puzzle page: exspectamus.dontstarvetogether.com.

### Open Developer Tools: on the Don't Starve Together puzzle page, press F12 or Ctrl + Shift + I.

### Open the console: click the Console tab.

### Load the script into the page: scroll down to the full code block, select all code and press Ctrl + C to copy it, return to the console, click the prompt, press Ctrl + V to paste the code, then press Enter to load it.

### Common commands:

- auto() - Run all remaining steps automatically.

- next() - Run one step.

- list() - View all puzzle steps.

- jumpTo(n) - Jump directly to step n.

- stop() - Emergency stop.

Note: do not use the mouse wheel to scroll the page while simulated dragging is running, or items may fly around unexpectedly.

```
// ==================== Don't Starve Together Auto Puzzle Solver - Ultimate Integrated Edition ====================
(function() {
'use strict';

// ==================== Automatic Page Navigation System ====================

// Define page transitions and click positions.
const PAGE_TRANSITIONS = {
    '0': {
        '1':  { panel: 0, x: 17, y: 15 }, // Sun switches to night.
        '78': { panel: 1, x: 14, y: 77 }  // Pencil switches to the study.
    },
    '1': {
        '0':  { panel: 0, x: 17, y: 15 }, // Moon switches to daytime.
        '4':  { panel: 0, x: 53, y: 85 }, // Dirt pit switches to the ruins.
        '2':  { panel: 16, x: 54, y: 39 } // Music-door note switches to the repair room.
    },
    '2': {
        '78': { panel: 0, x: 54, y: 44 }, // Crayon switches to the study.
        '5':  { panel: 0, x: 22, y: 46 }  // Door switches to the lens room.
    },
    '4': {
        '2':  { panel: 0, x: 21, y: 70 }  // Wagstaff part switches to the repair room.
    },
    '5': {
        '2':  { panel: 0, x: 22, y: 46 }  // Door switches to the repair room.
    },
    '78': {
        '1':  { panel: 0, x: 42, y: 89 }, // Coin switches to night.
        '2':  { panel: 0, x: 43, y: 62 }, // Screwdriver switches to the repair room.
        '4':  { panel: 0, x: 13, y: 32 }  // Part switches to the ruins.
    }
};

function getCurrentPage() {
    return document.body.getAttribute('page') || '0'; // Default to daytime (0) when no page is found.
}

// Core: use BFS to find the shortest transition path and click automatically.
async function navigateToPage(targetPage) {
    targetPage = String(targetPage);
    let currentPage = getCurrentPage();

    if (currentPage === targetPage) return;

    console.log(`\n🚗 Auto navigation: current Page ${currentPage}, target Page ${targetPage}`);

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
        console.error(`❌ Could not find a path from Page ${currentPage} to Page ${targetPage}. You may be in an unknown state.`);
        return;
    }

    // Follow the path and perform each transition.
    for (let i = 0; i < foundPath.length; i++) {
        let step = foundPath[i];
        console.log(`   [Page jump ${i + 1}/${foundPath.length}] from ${step.from} to ${step.to}`);

        await simulateClick({
            page: step.from,
            panel: step.action.panel,
            x: step.action.x,
            y: step.action.y,
            wait: 3500, // Allow time for page-jump animation/loading.
            desc: `Page route -> ${step.to}`,
            skipNav: true
        });

        let newPage = getCurrentPage();
        if (newPage !== step.to) {
             console.warn(`   ⚠️ The page does not seem to have updated; it is still ${newPage}. It may still be loading or a prerequisite may be locked.`);
        }
    }
    console.log(`✅ Navigation reached target Page ${getCurrentPage()}\n`);
}

// ==================== Utility And Feedback Functions ====================

function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

async function waitForElement(selector, timeout) {
    if (timeout === undefined) {
        // Special case: #page1panel6 may need up to three minutes while waiting for the cricket to appear.
        if (selector === '#page1panel6') {
            timeout = 180000;
            console.log(`⏳ [Hint] Waiting up to two minutes for the cricket/small curled mushroom. Please wait for the page to refresh.`);
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
    throw new Error(`Element not found: ${selector} (waited ${timeout / 1000} seconds)`);
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

// ==================== Smart Error-Prevention Guard System ====================

async function checkAndClosePanel(page, panel) {
    const pg = String(page);
    const pn = String(panel);

    if (pg === '78' && pn === '0') {
        let img = document.querySelector('#page78panel0 img.inspectable.open');
        if (img) {
            console.log("  => [Guard] Detected an unclosed item in page78panel0; resetting...");
            await simulateClick({page: 78, panel: 0, x: 0, y: 0, wait: 1000, skipNav: true, skipCheck: true});
        }
    } else if (pg === '0' && pn === '1') {
        let img = document.querySelector('#page0panel1 img.inspectable.open');
        if (img) {
            console.log("  => [Guard] Detected an unclosed item in page0panel1; resetting...");
            await simulateClick({page: 0, panel: 1, x: 0, y: 0, wait: 1000, skipNav: true, skipCheck: true});
        }
    } else if (pg === '5' && pn === '1') {
        if (document.getElementById('page5panel1openbook')) {
            console.log("  => [Guard] Detected an unclosed book in page5panel1; resetting...");
            await simulateClick({page: 5, panel: 1, x: 0, y: 0, wait: 1000, skipNav: true, skipCheck: true});
        }
    }
}

// ==================== Action Execution Engine ====================

async function simulateClick({page, panel, x, y, wait = 1000, desc = '', skipNav = false, skipCheck = false}) {
    if (!skipNav && page !== undefined) {
        await navigateToPage(page);
    }

    // Before clicking a panel, check whether any open items need to be closed.
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
    if (!element) return console.error(`  ✗ Element not found: ${sliderId}`);

    const isDraggablePanel = element.hasAttribute('candrag') && element.getAttribute('candrag') === 'true';
    const isDial = element.tagName.toLowerCase() === 'goggles-dial';
    const isMonocle = element.tagName.toLowerCase() === 'shadow-monocle';

    if (isDraggablePanel || isDial || isMonocle) {
        console.log(`  🔄 Drag ${sliderId}`);
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
        console.log(`  🔄 Slider ${sliderId}`);
        const back = element.querySelector('.rangeslider-back');
        const knob = element.querySelector('.rangeslider-knob');
        if (!back || !knob) return console.error(`  ✗ Slider component is missing required structure`);

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

// ==================== Smart Potion Helper Functions ====================

async function smartPreparePotion(recipeIds) {
    if (getCurrentPage() !== '1') await navigateToPage('1');

    // Smart cauldron detection.
    let panel20 = document.getElementById('page1panel20');
    if (!panel20) {
        console.log("  => Entering the cauldron interface...");
        await simulateClick({page: 1, panel: 16, x: 85, y: 62, wait: 5000, skipNav: true});
    }

    // Decide whether sap needs to be scooped. If liquid 984 does not exist, the vessel is empty.
    if (!document.getElementById('page1panel20-984')) {
        console.log("  => [Smart Init] Vessel is missing sap...");

        // Use the empty spoon to scoop sap.
        let emptySpoon = document.getElementById('page1panel20-477');
        if (emptySpoon) {
            await simulateDrag({page: 1, sliderId: 'page1panel20-477', startCoord: 'auto', endCoord: {x: 92, y: 67}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true});
        }

        // Pour the full spoon into the vessel.
        let fullSpoonId = document.getElementById('page1panel20-765') ? 'page1panel20-765' : 'page1panel20-477';
        await simulateDrag({page: 1, sliderId: fullSpoonId, startCoord: 'auto', endCoord: {x: 47, y: 43}, valueChange: {from: 0, to: 1}, wait: 3000, skipNav: true});
    }

    // Dynamic helper for collecting bugs from the bush.
    const waitAndClickBug = async (targetX, targetY) => {
        let panel6 = document.getElementById('page1panel6');
        if (!panel6) {
            console.log("  => [Helper] Clicking the bush. Please wait for the bug to crawl out...");
            await simulateClick({page: 1, panel: 7, x: 9, y: 70, wait: 2000, skipNav: true});
            await waitForElement('#page1panel6'); // Wait for panel 6 to appear.
        }
        await simulateClick({page: 1, panel: 6, x: targetX, y: targetY, wait: 2000, skipNav: true});
    };

    // Dictionary for all ingredients, collection methods, and initial center positions.
    const INGREDIENTS = {
        'page1panel20-948': { name: 'Mushroom (orange-white)', start: {x: 5.4, y: 66.85}, get: async () => { await simulateClick({page: 1, panel: 8, x: 36, y: 82, wait: 2000, skipNav: true}); } },
        'page1panel20-908': { name: 'Raspberry', start: {x: 29.4, y: 47.95}, get: async () => { await simulateClick({page: 1, panel: 4, x: 51, y: 52, wait: 2000, skipNav: true}); } },
        'page1panel20-712': { name: 'Mushroom (yellow curl)', start: {x: 20.3, y: 42.8}, get: async () => {
            await simulateClick({page: 1, panel: 7, x: 20, y: 80, wait: 2000, skipNav: true});
            await simulateClick({page: 1, panel: 7, x: 37, y: 70, wait: 2000, skipNav: true});
        } },
        'page1panel20-432': { name: 'Cricket', start: {x: 44.5, y: 66.45}, get: async () => { await waitAndClickBug(26, 77); } },
        'page1panel20-378': { name: 'Mushroom (red cap)', start: {x: 83.1, y: 72.6}, get: async () => { await simulateClick({page: 1, panel: 7, x: 80, y: 81, wait: 2000, skipNav: true}); } },
        'page1panel20-876': { name: 'Mushroom (small curl)', start: {x: 69.9, y: 62.85}, get: async () => { await waitAndClickBug(83, 84); } },
        'page1panel20-547': { name: 'Blueberry', start: {x: 24.55, y: 59.6}, get: async () => { await simulateClick({page: 1, panel: 12, x: 92, y: 12, wait: 2000, skipNav: true}); } },
        'page1panel20-602': { name: 'Bee', start: {x: 77.05, y: 35.7}, get: async () => { await simulateClick({page: 1, panel: 10, x: 65, y: 63, wait: 2000, skipNav: true}); } },
        'page1panel20-496': { name: 'Earthworm', start: {x: 5.6, y: 84.35}, get: async () => { await simulateClick({page: 1, panel: 8, x: 13, y: 82, wait: 2000, skipNav: true}); } },
    };

    // 2. Check whether required ingredients exist on screen; collect any missing ones.
    for (let id of recipeIds) {
        if (!document.getElementById(id)) {
            console.log(`  => ⚠ Missing recipe ingredient [${INGREDIENTS[id].name}], collecting automatically...`);
            await navigateToPage('1');
            await INGREDIENTS[id].get();
            console.log(`  => [${INGREDIENTS[id].name}] collected, returning to the cauldron...`);
            await navigateToPage('1');
            await simulateClick({page: 1, panel: 16, x: 85, y: 62, wait: 4000, skipNav: true});
        }
    }

    // 3. Reset the pestle so it does not block anything.
    let pestle = document.getElementById('page1panel20-271');
    if (pestle) {
        let px = (parseFloat(pestle.style.left) || 0) + (parseFloat(pestle.style.width) || 0) / 2;
        let py = (parseFloat(pestle.style.top) || 0) + (parseFloat(pestle.style.height) || 0) / 2;
        if (Math.sqrt(Math.pow(px - 62.5, 2) + Math.pow(py - 46.4, 2)) > 4) {
            console.log("  => [Smart Init] Resetting the pestle to avoid obstruction...");
            await simulateDrag({page: 1, sliderId: 'page1panel20-271', startCoord: 'auto', endCoord: {x: 62.5, y: 46.4}, valueChange: {from: 0, to: 1}, wait: 1200, skipNav: true});
        }
    }

    // Target center position for placing items in the vessel.
    const POT_CENTER = {x: 49, y: 38};

    const DOM_ORDER = [
        'page1panel20-547', // Blueberry
        'page1panel20-908', // Raspberry
        'page1panel20-602', // Bee
        'page1panel20-496', // Earthworm
        'page1panel20-432', // Cricket
        'page1panel20-876', // Small curl
        'page1panel20-378', // Red cap
        'page1panel20-712', // Yellow curl
        'page1panel20-948'  // Orange-white
    ];

    // 4. Scan all 9 ingredients and move any that are not in their target position.
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
                console.log(`  => [Potion] Moving required ingredient [${INGREDIENTS[id].name}] to the vessel center...`);
                await simulateDrag({page: 1, sliderId: id, startCoord: 'auto', endCoord: POT_CENTER, valueChange: {from: 0, to: 1}, wait: 1200, skipNav: true});
            }
        } else {
            if (!isAtStart) {
                console.log(`  => [Cleanup] Moving extra ingredient [${INGREDIENTS[id].name}] back to its initial position...`);
                await simulateDrag({page: 1, sliderId: id, startCoord: 'auto', endCoord: INGREDIENTS[id].start, valueChange: {from: 0, to: 1}, wait: 1200, skipNav: true});
            }
        }
    }

    // 5. Final pestle action.
    console.log("  => Start grinding! (move the pestle to x:54, y:28)");
    await simulateDrag({page: 1, sliderId: 'page1panel20-271', startCoord: 'auto', endCoord: {x: 54, y: 28}, valueChange: {from: 0, to: 1}, wait: 4000, skipNav: true});
}

// ==================== Full Action Table ====================

window.MANUAL = [
    // ========== Manual 1: Window And Hourglass ==========
    {name: "Window and Hourglass", actions: [
        {type: 'click', page: 0, panel: 0, x: 36, y: 44, wait: 3000},
        {type: 'click', page: 0, panel: 1, x: 80, y: 65, wait: 5000}
    ]},

    // ========== Manual 2: Click Manifestum To Get A New Image ==========
    {name: "Click Manifestum to Get a New Image", actions: [
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

    // ========== Manual 3: Click Wagstaff To Get A New Image ==========
    {name: "Click Wagstaff to Get a New Image", actions: [
        {type: 'click', page: 0, panel: 24, x: 46, y: 45, wait: 800},
        {type: 'click', page: 0, panel: 2, x: 26, y: 57, wait: 800},
        {type: 'click', page: 0, panel: 8, x: 62, y: 46, wait: 800},
        {type: 'click', page: 0, panel: 20, x: 34, y: 47, wait: 800},
        {type: 'click', page: 0, panel: 21, x: 36, y: 47, wait: 800},
        {type: 'click', page: 0, panel: 2, x: 55, y: 50, wait: 800},
        {type: 'click', page: 0, panel: 7, x: 33, y: 40, wait: 800},
        {type: 'click', page: 0, panel: 7, x: 33, y: 40, wait: 4000}
    ]},

    // ========== Manual 4: Click The Hourglass ==========
    {name: "Click the Hourglass", actions: [
        {type: 'click', page: 0, panel: 1, x: 82, y: 61, wait: 3000}
    ]},

    // ========== Manual 5: Pull Out The Note ==========
    {name: "Pull Out the Note", actions: [
        {type: 'click', page: 0, panel: 9, x: 61, y: 66, wait: 2000}
    ]},

    // ========== Manual 6: Connect The Wires ==========
    {name: "Connect the Wires", actions: [
        {type: 'click', page: 0, panel: 4, x: 19, y: 48, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 75, y: 64, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 46, y: 60, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 46, y: 60, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 75, y: 67, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 17, y: 52, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 78, y: 65, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 46, y: 56, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 16, y: 48, wait: 1000}, {type: 'click', page: 0, panel: 4, x: 16, y: 55, wait: 1000},
        {type: 'click', page: 0, panel: 4, x: 43, y: 60, wait: 4000}
    ]},

    // ========== Manual 7: Control Panel (Lever + Sliders + Buttons) ==========
    {name: "Control Panel (Lever + Sliders + Buttons)", actions: [
        // Dynamic init: replace the fuse and check the switch state.
        async function() {
            if (getCurrentPage() !== '0') await navigateToPage('0');
            console.log("  => [Init] Attempting to replace the fuse...");
            await simulateClick({page: 0, panel: 9, x: 37, y: 90, wait: 1000, skipNav: true});
            await simulateClick({page: 0, panel: 9, x: 40, y: 88, wait: 1000, skipNav: true});
            await simulateClick({page: 0, panel: 9, x: 40, y: 88, wait: 1000, skipNav: true});

            console.log("  => [Init] Checking switch state...");
            let switch1 = document.getElementById('panel9switch1');
            if (switch1 && switch1.classList.contains('toggled')) {
                console.log("  => [Init] Switch is already toggled; clicking it...");
                await simulateClick({page: 0, panel: 9, x: 13, y: 60, wait: 1000, skipNav: true});
            }
        },
        // Initial click to start the lever.
        {type: 'click', page: 0, panel: 9, x: 9, y: 42, wait: 1000},
        // First slider group.
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 56, y: 83}, endCoord: {x: 54, y: 68}, valueChange: {from: 0, to: 15.2299}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 31, y: 84}, endCoord: {x: 38, y: 76}, valueChange: {from: 0, to: 10.6658}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 50, y: 69}, endCoord: {x: 54, y: 34}, valueChange: {from: 15.2299, to: 38.0747}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 63, y: 78}, endCoord: {x: 52, y: 54}, valueChange: {from: 0, to: 22.8448}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 41, y: 74}, endCoord: {x: 44, y: 66}, valueChange: {from: 10.6658, to: 21.3316}, wait: 1500},
        // First button group.
        {type: 'click', page: 0, panel: 9, x: 59, y: 35, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 62, y: 37, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 58, y: 50, wait: 1000},
        // Second slider group.
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 43, y: 47}, endCoord: {x: 45, y: 53}, valueChange: {from: 38.0747, to: 22.8448}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 41, y: 63}, endCoord: {x: 46, y: 55}, valueChange: {from: 21.3316, to: 31.9974}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 52, y: 58}, endCoord: {x: 63, y: 95}, valueChange: {from: 22.8448, to: 0}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 45, y: 60}, endCoord: {x: 65, y: 95}, valueChange: {from: 22.8448, to: 0}, wait: 1500},
        // Second button group.
        {type: 'click', page: 0, panel: 9, x: 65, y: 40, wait: 1000},
        // Third slider group.
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 63, y: 81}, endCoord: {x: 61, y: 82}, valueChange: {from: 0, to: 7.61494}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 49, y: 55}, endCoord: {x: 59, y: 40}, valueChange: {from: 31.9974, to: 42.6632}, wait: 1500},
        // Third button group.
        {type: 'click', page: 0, panel: 9, x: 63, y: 52, wait: 1000},
        // Fourth slider group.
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 59, y: 75}, endCoord: {x: 50, y: 32}, valueChange: {from: 7.61494, to: 38.0747}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 65, y: 81}, endCoord: {x: 54, y: 60}, valueChange: {from: 0, to: 22.8448}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 49, y: 45}, endCoord: {x: 55, y: 27}, valueChange: {from: 42.6632, to: 53.329}, wait: 1500},
        // Fifth slider group.
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 47, y: 46}, endCoord: {x: 48, y: 49}, valueChange: {from: 38.0747, to: 30.4598}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 50, y: 60}, endCoord: {x: 52, y: 49}, valueChange: {from: 22.8448, to: 30.4598}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 51, y: 36}, endCoord: {x: 62, y: 12}, valueChange: {from: 53.329, to: 63.9948}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 52, y: 52}, endCoord: {x: 54, y: 65}, valueChange: {from: 30.4598, to: 15.2299}, wait: 1500},
        // Final button group.
        {type: 'click', page: 0, panel: 9, x: 35, y: 45, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 59, y: 50, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 80, y: 56, wait: 1000},
        {type: 'click', page: 0, panel: 9, x: 63, y: 51, wait: 1000},
        // Final slider action.
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 48, y: 68}, endCoord: {x: 48, y: 44}, valueChange: {from: 15.2299, to: 30.4598}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider2', startCoord: {x: 50, y: 55}, endCoord: {x: 57, y: 60}, valueChange: {from: 30.4598, to: 22.8448}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider1', startCoord: {x: 52, y: 53}, endCoord: {x: 45, y: 37}, valueChange: {from: 30.4598, to: 38.0747}, wait: 1500},
        {type: 'drag', page: 0, sliderId: 'panel9slider3', startCoord: {x: 56, y: 26}, endCoord: {x: 74, y: -12}, valueChange: {from: 63.9948, to: 74.6606}, wait: 5000}
    ]},

    // ========== Manual 8: Kick Items (5 Times) ==========
    {name: "Kick Items (5 Times)", actions: [
        {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 1000}, {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 1000},
        {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 1000}, {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 1000},
        {type: 'click', page: 0, panel: 26, x: 80, y: 70, wait: 4000}
    ]},

    // ========== Manual 9: New Image (Ear/Note/Tile) ==========
    {name: "New Image (Ear/Note/Tile)", actions: [
        {type: 'click', page: 0, panel: 33, x: 22, y: 49, wait: 1000},
        {type: 'click', page: 0, panel: 34, x: 53, y: 72, wait: 1000},
        {type: 'click', page: 0, panel: 35, x: 56, y: 55, wait: 4000}
    ]},

    // ========== Manual 10: Click Screws (4) ==========
    {name: "Click Screws (4)", actions: [
        {type: 'click', page: 0, panel: 38, x: 61, y: 94, wait: 800}, {type: 'click', page: 0, panel: 38, x: 82, y: 70, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 61, y: 74, wait: 800}, {type: 'click', page: 0, panel: 38, x: 82, y: 93, wait: 4000}
    ]},

    // ========== Manual 11: Click Small Buttons (11) ==========
    {name: "Click Small Buttons (11)", actions: [
        {type: 'click', page: 0, panel: 38, x: 76, y: 78, wait: 800}, {type: 'click', page: 0, panel: 38, x: 72, y: 79, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 72, y: 75, wait: 800}, {type: 'click', page: 0, panel: 38, x: 69, y: 76, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 69, y: 83, wait: 800}, {type: 'click', page: 0, panel: 38, x: 76, y: 83, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 76, y: 91, wait: 800}, {type: 'click', page: 0, panel: 38, x: 73, y: 92, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 72, y: 88, wait: 800}, {type: 'click', page: 0, panel: 38, x: 68, y: 88, wait: 800},
        {type: 'click', page: 0, panel: 38, x: 69, y: 92, wait: 4000}
    ]},

    // ========== Manual 12: Click Items ==========
    {name: "Click Items", actions: [
        {type: 'click', page: 1, panel: 0, x: 94, y: 83, wait: 1000}, {type: 'click', page: 1, panel: 1, x: 66, y: 36, wait: 1000},
        {type: 'click', page: 1, panel: 1, x: 62, y: 54, wait: 1000}, {type: 'click', page: 1, panel: 1, x: 78, y: 53, wait: 1000}
    ]},

    // ========== Manual 13: Sap Bucket ==========
    {name: "Sap Bucket", actions: [{type: 'click', page: 1, panel: 3, x: 40, y: 77, wait: 5000}]},

    // ========== Manual 14: Take The Key ==========
    {name: "Take the Key", actions: [
        async function() {
            if (getCurrentPage() !== '0') await navigateToPage('0');

            let drawer = document.getElementById('page0panel1drawer');
            if (!drawer) {
                console.log("  => Drawer is not open; trying to click it open...");
                await simulateClick({page: 0, panel: 1, x: 80, y: 91, wait: 2000, skipNav: true});
            }

            console.log("  => Taking the key...");
            await simulateClick({page: 0, panel: '1drawer', x: 47, y: 40, wait: 1500, skipNav: true});

            console.log("  => Closing the drawer...");
            await simulateClick({page: 0, panel: 1, x: 80, y: 91, wait: 1500, skipNav: true});
        }
    ]},

    // ========== Manual 15: Take The Notes ==========
    {name: "Take the Notes", actions: [
        async function() {
            if (getCurrentPage() !== '1') await navigateToPage('1');

            let toolbox = document.getElementById('page1panel4toolbox');
            if (!toolbox) {
                console.log("  => Toolbox is not open; trying to click it open...");
                await simulateClick({page: 1, panel: 4, x: 7, y: 35, wait: 2000, skipNav: true});
            }

            // 1. Check whether the lock has already been opened by looking for the three notes inside.
            const notes = ['page1panel4toolboxhatchnote', 'page1panel4toolboxbushnote', 'page1panel4toolboxdepositslip'];
            let hasNotes = notes.some(id => document.getElementById(id) !== null);

            if (hasNotes) {
                console.log("  => [Smart Check] Notes found inside the lock, so the key has already opened it. Dragging notes out directly...");
                for (let noteId of notes) {
                    if (document.getElementById(noteId)) {
                        console.log(`  => 📄 Dragging note ${noteId} out...`);
                        await simulateDrag({ page: 1, sliderId: noteId, startCoord: 'auto', endCoord: {x: 94, y: 40}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true });
                    }
                }
                console.log("  => ✅ Operation finished; closing the toolbox directly.");
                await simulateClick({page: 1, panel: 4, x: 7, y: 35, wait: 1000, skipNav: true});
                return; // Finish early.
            }

            // 2. If no notes were found, check the toolbox panel and key state.
            let panel = document.getElementById('page1panel4toolboxpanel');
            if (panel) {
                let key = document.getElementById('page1panel4toolboxkey');
                if (!key) {
                    console.log("  => ⚠ Key is missing. Running automatic recovery: go back and take the key...");
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
                        console.log("  => Moving the hammer away...");
                        await simulateDrag({ page: 1, sliderId: 'page1panel4toolboxhammer', startCoord: 'auto', endCoord: {x: 62, y: 54}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true });
                    }
                    console.log("  => Moving the wooden board away...");
                    await simulateDrag({ page: 1, sliderId: 'page1panel4toolboxpanel', startCoord: 'auto', endCoord: {x: 13, y: 73}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true });

                    console.log("  => 🔑 Key position detected; dragging it to unlock...");
                    await simulateDrag({ page: 1, sliderId: 'page1panel4toolboxkey', startCoord: 'auto', endCoord: {x: 33, y: 74}, valueChange: {from: 0, to: 1}, wait: 2000, skipNav: true });

                    for (let noteId of notes) {
                        if (document.getElementById(noteId)) {
                            console.log(`  => 📄 Dragging note ${noteId} out...`);
                            await simulateDrag({ page: 1, sliderId: noteId, startCoord: 'auto', endCoord: {x: 94, y: 40}, valueChange: {from: 0, to: 1}, wait: 1500, skipNav: true });
                        }
                    }

                    console.log("  => ✅ Operation finished; closing the toolbox directly.");
                    await simulateClick({page: 1, panel: 4, x: 7, y: 35, wait: 1000, skipNav: true});
                }
            } else {
                console.log("  => ✅ Toolbox is already empty; closing it directly.");
                await simulateClick({page: 1, panel: 4, x: 7, y: 35, wait: 1000, skipNav: true});
            }
        }
    ]},

    // ========== Manual 16: Berry Bush ==========
    {name: "Berry Bush", actions: [
        {type: 'click', page: 1, panel: 11, x: 69, y: 57, wait: 1000}, {type: 'click', page: 1, panel: 11, x: 77, y: 76, wait: 1000},
        {type: 'click', page: 1, panel: 11, x: 94, y: 76, wait: 1000}, {type: 'click', page: 1, panel: 11, x: 86, y: 56, wait: 1000},
        {type: 'click', page: 1, panel: 11, x: 55, y: 76, wait: 1000}
    ]},

    // ========== Manual 17: Click The Bush To Find The Cricket ==========
    {name: "Click the Bush to Find the Cricket", actions: [
        async function() {
            let panel6 = document.getElementById('page1panel6');
            if (!panel6) {
                console.log("  => Clicking the bush. Please wait for the bug to appear...");
                await simulateClick({page: 1, panel: 7, x: 9, y: 70, wait: 5000, skipNav: true});
            }
        }
    ]},

    // ========== Manual 18: Cricket And Mushrooms ==========
    {name: "Cricket and Mushrooms", actions: [
        async function() {
            let panel6 = document.getElementById('page1panel6');
            if (!panel6) {
                console.log("  => [Collection Helper] Bug did not appear; trying to click the bush...");
                await simulateClick({page: 1, panel: 7, x: 9, y: 70, wait: 2000, skipNav: true});
                await waitForElement('#page1panel6');
            }
            await simulateClick({page: 1, panel: 6, x: 26, y: 77, wait: 800, skipNav: true});
            await simulateClick({page: 1, panel: 6, x: 83, y: 84, wait: 800, skipNav: true});
        }
    ]},

    // ========== Manual 19: Collect Various Ingredients ==========
    {name: "Collect Various Ingredients", actions: [
        {type: 'click', page: 1, panel: 4, x: 51, y: 52, wait: 1000}, {type: 'click', page: 1, panel: 5, x: 7, y: 16, wait: 1000},
        {type: 'click', page: 1, panel: 5, x: 37, y: 89, wait: 1000}, {type: 'click', page: 1, panel: 7, x: 20, y: 80, wait: 1000},
        {type: 'click', page: 1, panel: 7, x: 37, y: 70, wait: 1000}, {type: 'click', page: 1, panel: 7, x: 80, y: 81, wait: 1000},
        {type: 'click', page: 1, panel: 8, x: 13, y: 82, wait: 1000}, {type: 'click', page: 1, panel: 8, x: 36, y: 82, wait: 1000},
        {type: 'click', page: 1, panel: 10, x: 32, y: 85, wait: 1000}, {type: 'click', page: 1, panel: 10, x: 65, y: 63, wait: 1000},
        {type: 'click', page: 1, panel: 12, x: 92, y: 12, wait: 4000}
    ]},

    // ========== Manual 20: Open The Manhole Cover And Enter ==========
    {name: "Open the Manhole Cover and Enter", actions: [
        {type: 'click', page: 1, panel: 12, x: 63, y: 70, wait: 800}, {type: 'click', page: 1, panel: 12, x: 63, y: 70, wait: 800},
        {type: 'click', page: 1, panel: 12, x: 58, y: 81, wait: 800}, {type: 'click', page: 1, panel: 12, x: 63, y: 70, wait: 800},
        {type: 'click', page: 1, panel: 12, x: 70, y: 81, wait: 800}, {type: 'click', page: 1, panel: 12, x: 58, y: 78, wait: 800},
        {type: 'click', page: 1, panel: 12, x: 58, y: 78, wait: 800}, {type: 'click', page: 1, panel: 12, x: 63, y: 70, wait: 800},
        {type: 'click', page: 1, panel: 12, x: 71, y: 79, wait: 4000},
        {type: 'click', page: 1, panel: 13, x: 42, y: 70, wait: 2000}
    ]},

    // ========== Manual 21: Potion Recipe One ==========
    {name: "Potion Recipe One", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-948', 'page1panel20-908', 'page1panel20-712', 'page1panel20-432']);
        }
    ]},

    // ========== Manual 22: Potion Recipe Two ==========
    {name: "Potion Recipe Two", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-378', 'page1panel20-908', 'page1panel20-712', 'page1panel20-432']);
        }
    ]},

    // ========== Manual 23: Potion Recipe Three ==========
    {name: "Potion Recipe Three", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-876', 'page1panel20-908', 'page1panel20-712', 'page1panel20-432']);
        }
    ]},

    // ========== Manual 24: Potion Recipe Four ==========
    {name: "Potion Recipe Four", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-876', 'page1panel20-547', 'page1panel20-712', 'page1panel20-602', 'page1panel20-948']);
        }
    ]},

    // ========== Manual 25: Potion Recipe Five ==========
    {name: "Potion Recipe Five", actions: [
        async function() {
            await smartPreparePotion(['page1panel20-908', 'page1panel20-432', 'page1panel20-547', 'page1panel20-496']);
        }
    ]},

    // ========== Manual 26: Open The Music Door ==========
    {name: "Open the Music Door", actions: [
        async function() {
            if (getCurrentPage() !== '1') await navigateToPage('1');

            // Check whether the music door is already open by looking for the small note on the door.
            let screwdriver = document.getElementById('page1panel16screwdriver');
            if (screwdriver) {
                console.log("  => [Smart Check] Small note found. The music door is already open, skipping this puzzle step.");
                return;
            }

            console.log("  => Clicking the music bell to activate the door...");
            await simulateClick({page: 1, panel: 16, x: 17, y: 50, wait: 2000, skipNav: true});

            console.log("  => Clicking the music door in sequence...");
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

    // ========== Manual 27: Click Vestigo To Reveal The Dirt Pit ==========
    {name: "Click Vestigo to Reveal the Dirt Pit", actions: [
        {type: 'click', page: 1, panel: 0, x: 56, y: 77, wait: 1000}, {type: 'click', page: 1, panel: 0, x: 35, y: 57, wait: 1000},
        {type: 'click', page: 1, panel: 0, x: 45, y: 76, wait: 1000}, {type: 'click', page: 1, panel: 0, x: 43, y: 56, wait: 1000},
        {type: 'click', page: 1, panel: 0, x: 64, y: 77, wait: 1000}, {type: 'click', page: 1, panel: 0, x: 25, y: 77, wait: 1000},
        {type: 'click', page: 1, panel: 0, x: 27, y: 58, wait: 4000}
    ]},

    // ========== Manual 28: Focus The Goggles ==========
    {name: "Focus the Goggles", actions: [
        {type: 'click', page: 4, panel: 0, x: 26, y: 28, wait: 5000},
        {type: 'drag', page: 4, sliderId: 'page4panel1dial1', startCoord: {x: 27, y: 9}, endCoord: {x: 24, y: 10}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page4panel1dial2', startCoord: {x: 36, y: 76}, endCoord: {x: 31, y: 79}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page4panel1dial3', startCoord: {x: 71, y: 10}, endCoord: {x: 78, y: 11}, valueChange: {from: 0, to: 1}, wait: 1500},
        {type: 'drag', page: 4, sliderId: 'page4panel1dial4', startCoord: {x: 70, y: 83}, endCoord: {x: 78, y: 81}, valueChange: {from: 0, to: 1}, wait: 3000}
    ]},

    // ========== Manual 29: Find Parts (5) ==========
    {name: "Find Parts (5)", actions: [
        {type: 'click', page: 4, panel: 2, x: 16, y: 18, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 33, y: 66, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 71, y: 47, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 40, y: 75, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 86, y: 47, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 33, y: 34, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 30, y: 75, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 24, y: 48, wait: 1000},
        {type: 'click', page: 4, panel: 2, x: 27, y: 41, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 40, y: 82, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 63, y: 48, wait: 1000}, {type: 'click', page: 4, panel: 2, x: 42, y: 63, wait: 1000},
        {type: 'click', page: 4, panel: 3, x: 50, y: 96, wait: 1000}, {type: 'click', page: 4, panel: 3, x: 20, y: 51, wait: 1000}
    ]},

    // ========== Manual 30: Get The Gramophone ==========
    {name: "Get the Gramophone", actions: [
        {type: 'click', page: 2, panel: 0, x: 5, y: 49, wait: 3000},
        {type: 'click', page: 2, panel: 1, x: 78, y: 43, wait: 3000}
    ]},

    // ========== Manual 31: Adjust The Laser Lenses ==========
    {name: "Adjust the Laser Lenses", actions: [
        // Dynamic init: open the filter window, remove interfering beams, then move the control bar down to the bottom.
        async function() {
            if (getCurrentPage() !== '5') await navigateToPage('5');

            let panel1 = document.getElementById('page5panel1');
            if (!panel1) {
                console.log("  => [Init] Filter window is not open; trying to click it open...");
                await simulateClick({page: 5, panel: 0, x: 82, y: 61, wait: 4000, skipNav: true});
            }

            // Only check for unclosed boxes during initialization.
            if (document.getElementById('page5panel1openbox')) {
                console.log("  => [Init] Unclosed box found; performing one reset...");
                await simulateClick({page: 5, panel: 1, x: 0, y: 0, wait: 1000, skipNav: true, skipCheck: true});
            }

            console.log("  => [Init] 1. Clearing all filters...");
            await simulateClick({page: 5, panel: 1, x: 11, y: 73, wait: 1500, skipNav: true});

            console.log("  => [Init] 2. Checking slider position...");
            let indicator = document.getElementById('page5panel1slotindicator');
            let maxMoves = 30;
            while (indicator && !indicator.style.top.includes('77') && maxMoves > 0) {
                console.log(`  => Current slider position: ${indicator.style.top}; moving further down...`);
                await simulateClick({page: 5, panel: 1, x: 91, y: 90, wait: 1000, skipNav: true});
                maxMoves--;
            }
            if (maxMoves === 0) {
                console.warn("  => [Warning] Slider movement seems to have reached its limit or the target position was not found; continuing.");
            }

            console.log("  => [Init] 3. Checking for existing beams...");
            const beams = [
                { id: 'page5panel1lbeam', x: 36, y: 94 },
                { id: 'page5panel1mbeam', x: 46, y: 94 },
                { id: 'page5panel1hbeam', x: 56, y: 94 },
                { id: 'page5panel1abeam', x: 66, y: 94 }
            ];
            for (let beam of beams) {
                if (document.getElementById(beam.id)) {
                    console.log(`  => Found existing ${beam.id}; clearing it...`);
                    await simulateClick({page: 5, panel: 1, x: beam.x, y: beam.y, wait: 1500, skipNav: true});
                }
            }
            console.log("  => [Init Complete] Starting the main lens adjustment sequence.");
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

    // ========== Manual 32: Get The Lens ==========
    {name: "Get the Lens", actions: [
        {type: 'drag', page: 4, sliderId: 'page4panel1dial5', startCoord: {x: 9, y: 60}, endCoord: {x: 12, y: 70}, valueChange: {from: 0, to: 1}, wait: 3000}
    ]},

    // ========== Manual 33: Find Petals (17) ==========
    {name: "Find Petals (17)", actions: [
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

    // ========== Manual 34: Find Remaining Parts (35) ==========
    {name: "Find Remaining Parts (35)", actions: [
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

    // ========== Manual 35: Tuning Fork Puzzle ==========
    {name: "Tuning Fork Puzzle", actions: [
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

// ==================== Scheduler Controller ====================

const Controller = {
    config: window.MANUAL,
    currentStage: 0,
    isRunning: false,
    autoMode: false,

    list() {
        if (!this.config) return console.log('❌ Configuration not found');
        console.log(`\n${'━'.repeat(50)}\n📋 Action step list:\n${'━'.repeat(50)}`);
        this.config.forEach((stage, idx) => {
            console.log(`${idx + 1}. ${stage.name}`);
        });
        console.log(`${'━'.repeat(50)}\n💡 Enter jumpTo(n) to jump to stage n\n`);
    },

    async executeStage(index) {
        if (!this.config || index < 0 || index >= this.config.length) return false;

        this.isRunning = true;
        const stage = this.config[index];

        console.log(`\n${'='.repeat(50)}\nRunning stage ${index + 1}/${this.config.length}: ${stage.name}\n${'='.repeat(50)}`);

        try {
            for (let i = 0; i < stage.actions.length; i++) {
                if (!this.isRunning) return false;
                console.log(`[Step ${i + 1}/${stage.actions.length}]`);

                const action = stage.actions[i];

                if (typeof action === 'function') {
                    await action();
                } else if (action.type === 'click') {
                    await simulateClick(action);
                } else if (action.type === 'drag') {
                    await simulateDrag(action);
                }
            }
            console.log(`✓ Stage ${index + 1} complete`);
            this.isRunning = false;
            return true;
        } catch (error) {
            if (error.message && error.message.includes("Element not found")) {
                console.error(`❌ Execution stopped: page element not found.\n\n💡 Hint: ${error.message}.\nMake sure prerequisite puzzle steps are complete, or try again after the network finishes loading. You can also try skipping the current step with jumpTo.`);
            } else {
                console.error(`❌ Execution error:`, error);
            }
            this.isRunning = false;
            return false;
        }
    },

    async next() {
        if (this.isRunning) return console.warn('⚠️ Execution is already running. Please wait for it to finish.');
        if (this.currentStage >= this.config.length) return console.log('\n🎉 All stages are complete!');

        const success = await this.executeStage(this.currentStage);
        if (success) {
            this.currentStage++;
            if (this.currentStage < this.config.length) {
                console.log(`💡 Enter next() to continue to the next stage`);
                if (this.autoMode) {
                    await sleep(1000);
                    await this.next();
                }
            } else {
                console.log('\n🎊 All story content has been unlocked. Puzzle complete!');
            }
        }
    },

    async auto() {
        console.log(`\n🚀 Starting full auto-run mode! Total stages: ${this.config.length}.\n`);
        this.autoMode = true;
        await this.next();
    },

    jumpTo(stageNumber) {
        const index = stageNumber - 1;
        if (index < 0 || index >= this.config.length) return console.error(`❌ Invalid number. Please enter 1-${this.config.length}`);

        this.currentStage = index;
        console.log(`\n${'━'.repeat(50)}\n📍 Jumped to stage ${stageNumber}/${this.config.length} => [${this.config[index].name}]\n💡 Enter next() to start execution.\n(⚠️ The script will automatically navigate to the required page while running.)\n${'━'.repeat(50)}`);
    },

    stop() {
        this.isRunning = false;
        this.autoMode = false;
        console.log('🛑 Execution stopped');
    }
};

window.list = () => Controller.list();
window.next = () => Controller.next();
window.auto = () => Controller.auto();
window.jumpTo = (n) => Controller.jumpTo(n);
window.stop = () => Controller.stop();

console.log(`
${'━'.repeat(60)}
🎮 Don't Starve Together Puzzle Automation Script - Ultimate Integrated Edition 🎮
${'━'.repeat(60)}
✅ Automatic navigation system loaded
✅ Smart protection logic loaded, including special handling for unclosed items
✅ Merged ${window.MANUAL.length} puzzle stages

[Command List]
  auto()      - Complete automatically
  next()      - Run one step at a time
  list()      - View the full action step list
  jumpTo(n)   - Jump to stage n (with automatic cross-page navigation)
  stop()      - Stop immediately
${'━'.repeat(60)}
`);

})();
```
