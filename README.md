// ==UserScript==
// @name         ChatGPT 🔥本気で聞くボタン
// @namespace    https://chatgpt.com/
// @version      1.1.0
// @description  添付送信時は通常Chatを最速、Workを軽量へ自動変更。本気ボタンは最上位設定で送信し、回答完了後に元へ戻します。
// @author       ebi
// @match        https://chatgpt.com/*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(() => {
    'use strict';

    const SCRIPT_NAME = 'ChatGPT 本気で聞く';
    const SCRIPT_VERSION = 'N1.1.0';
    const BUTTON_ID = 'ebi-honki-submit-button';
    let DEBUG_STAGE = '初期化';
    let attachmentSubmitInProgress = false;

    /*
     * 最大・非常に高いが利用可能な環境（Work）:
     *   最大 → 非常に高い → 高い
     *
     * 通常版 GPT-5.5以上:
     *   高い
     *
     * 現在が最速・中程度でも、送信後は元の設定へ戻す。
     *
     * GPT-5.3以前など高いが利用できない環境:
     *   送信せず、GPT-5.5以上にしてねと表示
     */
    const EFFORT_LEVELS = [
        {
            key: 'max',
            displayName: '最大',
            labels: ['最大', 'Max']
        },
        {
            key: 'extra-high',
            displayName: '非常に高い',
            labels: ['非常に高い', 'Extra High', 'Very High']
        },
        {
            key: 'high',
            displayName: '高い',
            labels: ['高い', 'High']
        },
        {
            key: 'medium',
            displayName: '中程度',
            labels: ['中程度', 'Medium']
        },
        {
            key: 'low',
            displayName: '低い',
            labels: ['軽', '低い', 'Low', 'Light']
        },
        {
            key: 'fast',
            displayName: '最速',
            labels: ['最速', 'Fast', 'Auto']
        }
    ];

    const TARGET_PRIORITY_WORK = ['max', 'extra-high', 'high'];
    const TARGET_PRIORITY_NORMAL = ['high'];
    const ATTACHMENT_PRIORITY_WORK = ['low'];
    const ATTACHMENT_PRIORITY_NORMAL = ['fast'];

    const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));

    const normalizeText = value =>
        String(value || '')
            .replace(/\s+/g, ' ')
            .trim();

    // 通常運用ではデバッグログを出さない。必要時は関数内を有効化する。
    function log(...args) {
        // console.log(`[${SCRIPT_NAME} v${SCRIPT_VERSION}]`, ...args);
    }

    function warn(...args) {
        // console.warn(`[${SCRIPT_NAME} v${SCRIPT_VERSION}]`, ...args);
    }

    function setDebugStage(stage, detail = null) {
        DEBUG_STAGE = stage;
        // log(`=== STAGE: ${stage} ===`, detail || '');
    }

    function summarizeMenu(menu) {
        if (!menu) return null;
        return {
            id: menu.id || null,
            role: menu.getAttribute('role'),
            state: menu.getAttribute('data-state'),
            visible: isVisible(menu),
            testid: menu.getAttribute('data-testid'),
            text: normalizeText(menu.innerText || menu.textContent).slice(0, 800),
            menuitems: [...menu.querySelectorAll('[role="menuitem"], [role="menuitemradio"]')].map(item => ({
                role: item.getAttribute('role'),
                text: normalizeText(item.innerText || item.textContent),
                visible: isVisible(item),
                expanded: item.getAttribute('aria-expanded'),
                controls: item.getAttribute('aria-controls'),
                checked: item.getAttribute('aria-checked'),
                state: item.getAttribute('data-state')
            }))
        };
    }

    function dumpOpenMenus(label = 'DOMスナップショット') {
        // デバッグ時のみ有効化する。
    }

    // 必要時にコンソールから呼び出せるが、通常版では何も出力しない。
    window.ebiHonkiDebugDump = () => dumpOpenMenus('手動debug dump');

    // log('loaded');

    function isVisible(element) {
        if (!(element instanceof Element)) return false;

        const style = window.getComputedStyle(element);
        const rect = element.getBoundingClientRect();

        return (
            style.display !== 'none' &&
            style.visibility !== 'hidden' &&
            Number(style.opacity || 1) !== 0 &&
            rect.width > 0 &&
            rect.height > 0
        );
    }

    async function waitFor(callback, timeoutMs = 10000, intervalMs = 100) {
        const startedAt = Date.now();

        while (Date.now() - startedAt < timeoutMs) {
            const result = callback();

            if (result) {
                return result;
            }

            await sleep(intervalMs);
        }

        throw new Error('待機時間内に操作対象を確認できませんでした');
    }

    function dispatchPointerClick(element) {
        if (!(element instanceof Element)) {
            throw new Error('クリック対象が見つかりません');
        }

        const rect = element.getBoundingClientRect();
        const common = {
            bubbles: true,
            cancelable: true,
            composed: true,
            clientX: rect.left + rect.width / 2,
            clientY: rect.top + rect.height / 2,
            button: 0,
            buttons: 1
        };

        element.focus?.({ preventScroll: true });
        element.dispatchEvent(new PointerEvent('pointerdown', {
            ...common,
            pointerId: 1,
            pointerType: 'mouse',
            isPrimary: true
        }));
        element.dispatchEvent(new MouseEvent('mousedown', common));
        element.dispatchEvent(new PointerEvent('pointerup', {
            ...common,
            buttons: 0,
            pointerId: 1,
            pointerType: 'mouse',
            isPrimary: true
        }));
        element.dispatchEvent(new MouseEvent('mouseup', {
            ...common,
            buttons: 0
        }));
        element.dispatchEvent(new MouseEvent('click', {
            ...common,
            buttons: 0,
            detail: 1
        }));
    }

    function getClickables(root = document) {
        return [...root.querySelectorAll(
            'button, [role="button"], [role="combobox"], ' +
            '[role="menuitem"], [role="option"], [role="radio"], ' +
            '[role="menuitemradio"], [aria-haspopup], [tabindex="0"]'
        )].filter(isVisible);
    }

    function textContainsLabel(text, label) {
        const normalizedText = normalizeText(text).toLocaleLowerCase();
        const normalizedLabel = normalizeText(label).toLocaleLowerCase();

        return Boolean(
            normalizedText &&
            normalizedLabel &&
            normalizedText.includes(normalizedLabel)
        );
    }

    function getLevelByKey(key) {
        return EFFORT_LEVELS.find(level => level.key === key) || null;
    }

    function inferLevelFromText(text) {
        /*
         * 「非常に高い」を「高い」より先に判定するため、
         * すべての表示名を文字数の長い順に調べます。
         */
        const entries = EFFORT_LEVELS
            .flatMap(level =>
                level.labels.map(label => ({
                    level,
                    label
                }))
            )
            .sort((a, b) => b.label.length - a.label.length);

        for (const entry of entries) {
            if (textContainsLabel(text, entry.label)) {
                return entry.level;
            }
        }

        return null;
    }

    function inferModelVersionFromText(text) {
        const normalized = normalizeText(text);

        /*
         * GPT-5.6 / 5.6 Sol / GPT 5.5 などを拾う。
         */
        const match = normalized.match(
            /(?:GPT[-\s]*)?5\.(\d+)/i
        );

        if (!match) {
            return null;
        }

        const minor = Number(match[1]);

        if (!Number.isFinite(minor)) {
            return null;
        }

        return {
            major: 5,
            minor,
            isAtLeast55: minor >= 5,
            raw: match[0]
        };
    }

    /*
     * 選択モデルの判定専用。
     * 「最速 5.5」のような応答性能側の補助数字はモデル名にしません。
     * GPT表記、またはSol/Work表記がある文字列だけを対象にします。
     */
    function inferExplicitModelVersionFromText(text) {
        const normalized = normalizeText(text);

        let match = normalized.match(
            /GPT[-\s]*5\.(\d+)/i
        );

        if (
            !match &&
            (
                /\bSol\b/i.test(normalized) ||
                /\bWork\b/i.test(normalized) ||
                /ワーク/.test(normalized)
            )
        ) {
            match = normalized.match(/5\.(\d+)/i);
        }

        if (!match) {
            return null;
        }

        const minor = Number(match[1]);

        if (!Number.isFinite(minor)) {
            return null;
        }

        return {
            major: 5,
            minor,
            isAtLeast55: minor >= 5,
            raw: match[0]
        };
    }

    function looksLikeModelOrEffortControlText(text) {
        const normalized = normalizeText(text);

        return Boolean(
            inferLevelFromText(normalized) ||
            inferModelVersionFromText(normalized) ||
            /Sol/i.test(normalized) ||
            /GPT[-\s]*5/i.test(normalized)
        );
    }

    function isWorkOrSolText(text) {
        const normalized = normalizeText(text);

        return /Sol/i.test(normalized) ||
               /\bWork\b/i.test(normalized) ||
               /ワーク/.test(normalized);
    }

    function findEditor() {
        const editors = [
            ...document.querySelectorAll(
                'textarea, [contenteditable="true"][role="textbox"], ' +
                '[contenteditable="true"]'
            )
        ].filter(isVisible);

        return editors.sort((a, b) => {
            const rectA = a.getBoundingClientRect();
            const rectB = b.getBoundingClientRect();

            const scoreA = rectA.top * 10 + rectA.width * rectA.height;
            const scoreB = rectB.top * 10 + rectB.width * rectB.height;

            return scoreB - scoreA;
        })[0] || null;
    }

    function getEditorText(editor) {
        if (!editor) return '';

        if (editor instanceof HTMLTextAreaElement) {
            return editor.value;
        }

        return editor.innerText || editor.textContent || '';
    }

    function editorHasText(editor) {
        return normalizeText(getEditorText(editor)).length > 0;
    }

    function getComposerRoot(editor = findEditor()) {
        if (!editor) return null;

        return (
            editor.closest('form') ||
            editor.closest('[data-type="unified-composer"]') ||
            editor.closest('[data-testid*="composer"]') ||
            editor.parentElement?.parentElement?.parentElement ||
            null
        );
    }


    const ATTACHMENT_FILE_PATTERN = /\.(?:txt|log|csv|json|xml|md|pdf|docx?|xlsx?|pptx?|zip|rar|7z|tar|gz|png|jpe?g|gif|webp|bmp|svg|mp3|wav|m4a|mp4|mov|avi|webm|cs|js|jsx|ts|tsx|py|java|cpp|c|h|html?|css|sql|obj|mtl)$/i;

    function getAttachmentElements(editor = findEditor()) {
        const composer = getComposerRoot(editor);
        if (!composer) return [];

        return [...composer.querySelectorAll('button[aria-label], [role="button"][aria-label]')]
            .filter(element => {
                const label = normalizeText(element.getAttribute('aria-label'));
                return label && ATTACHMENT_FILE_PATTERN.test(label);
            });
    }

    function composerHasAttachment(editor = findEditor()) {
        return getAttachmentElements(editor).length > 0;
    }

    function findSendButton(editor = findEditor()) {
        const selectors = [
            'button[data-testid="send-button"]',
            'button[aria-label*="送信"]',
            'button[aria-label*="Send"]',
            'button[title*="送信"]',
            'button[title*="Send"]'
        ].join(',');

        const roots = [
            editor?.closest('form'),
            getComposerRoot(editor),
            document
        ].filter(Boolean);

        for (const root of roots) {
            const button = [...root.querySelectorAll(selectors)]
                .find(candidate =>
                    isVisible(candidate) &&
                    candidate.id !== BUTTON_ID
                );

            if (button) return button;
        }

        return null;
    }

    function findStopButton() {
        const selectors = [
            'button[data-testid="stop-button"]',
            'button[aria-label*="停止"]',
            'button[aria-label*="中止"]',
            'button[aria-label*="Stop"]',
            'button[title*="停止"]',
            'button[title*="Stop"]'
        ].join(',');

        return [...document.querySelectorAll(selectors)]
            .find(isVisible) || null;
    }

    function countAssistantMessages() {
        return document.querySelectorAll(
            '[data-message-author-role="assistant"]'
        ).length;
    }

    function getLatestAssistantText() {
        const messages = [
            ...document.querySelectorAll(
                '[data-message-author-role="assistant"]'
            )
        ];

        const latest = messages[messages.length - 1];
        return normalizeText(latest?.innerText || latest?.textContent || '');
    }

    function getComposerSurface(editor = findEditor()) {
        return (
            editor?.closest('[data-composer-surface="true"]') ||
            editor?.closest('form') ||
            getComposerRoot(editor) ||
            null
        );
    }

    function getVisibleEffortTextFromButton(button) {
        if (!button) return '';

        /*
         * Workのボタンには、幅計測用として全モデル・全レベルの文字列が
         * aria-hidden=true で入っています。button.textContent 全体を読むと
         * 現在値ではない「最大」等を誤検出するため、表示中ラベルを優先します。
         */
        const visibleLabel = button.querySelector(
            '.uFxlGa_SliderTriggerContent .uFxlGa_SliderTriggerEffortLabel, ' +
            '[data-animated-slider-trigger="true"] .uFxlGa_SliderTriggerEffortLabel:not([aria-hidden="true"])'
        );

        if (visibleLabel && isVisible(visibleLabel)) {
            return normalizeText(visibleLabel.textContent);
        }

        const clone = button.cloneNode(true);
        clone.querySelectorAll('[aria-hidden="true"], .uFxlGa_SliderTriggerMeasurement')
            .forEach(element => element.remove());

        return normalizeText(clone.textContent);
    }

    function findEffortButton() {
        const editor = findEditor();
        const composer = getComposerSurface(editor);
        const sendButton = findSendButton(editor);

        if (!composer) return null;

        const candidates = [...composer.querySelectorAll(
            'button[aria-haspopup="menu"], [role="button"][aria-haspopup="menu"]'
        )]
            .filter(element => {
                if (
                    !isVisible(element) ||
                    element.id === BUTTON_ID ||
                    element === sendButton
                ) {
                    return false;
                }

                /* Workの新UIはこの専用トリガーで確実に判定できます。 */
                if (element.querySelector('[data-animated-slider-trigger="true"]')) {
                    return true;
                }

                return Boolean(inferLevelFromText(
                    getVisibleEffortTextFromButton(element)
                ));
            })
            .map(element => ({
                element,
                level: inferLevelFromText(
                    getVisibleEffortTextFromButton(element)
                ),
                distance: sendButton
                    ? Math.abs(
                        element.getBoundingClientRect().right -
                        sendButton.getBoundingClientRect().left
                    )
                    : 0
            }))
            .sort((a, b) => a.distance - b.distance);

        const winner = candidates[0]?.element || null;

        if (winner) {
            log('応答性能ボタン:', getVisibleEffortTextFromButton(winner));
        }

        return winner;
    }

    function detectCurrentEffortLevel() {
        const button = findEffortButton();
        return button
            ? inferLevelFromText(getVisibleEffortTextFromButton(button))
            : null;
    }

    function detectCurrentEnvironment() {
        const button = findEffortButton();
        const buttonText = getVisibleEffortTextFromButton(button);
        const fullButtonText = normalizeText(button?.textContent);

        return {
            button,
            buttonText,
            currentLevel: button
                ? inferLevelFromText(buttonText)
                : null,
            modelVersion: button
                ? inferExplicitModelVersionFromText(fullButtonText)
                : null
        };
    }

    function closeEffortMenu() {
        const makeEvent = type => new KeyboardEvent(type, {
            key: 'Escape',
            code: 'Escape',
            keyCode: 27,
            which: 27,
            bubbles: true,
            cancelable: true,
            composed: true
        });

        /* Workは親メニュー＋思考レベルのサブメニュー。余裕を持って3段閉じる。 */
        for (let i = 0; i < 3; i++) {
            const openMenu = [...document.querySelectorAll(
                '[role="menu"][data-state="open"], [role="menu"]'
            )].filter(isVisible).at(-1);
            const target = openMenu || document.activeElement || document;

            target.dispatchEvent(makeEvent('keydown'));
            target.dispatchEvent(makeEvent('keyup'));
            document.dispatchEvent(makeEvent('keydown'));
            document.dispatchEvent(makeEvent('keyup'));
        }
    }

    async function closeEffortMenuFully() {
        closeEffortMenu();
        await sleep(120);
        closeEffortMenu();
        await sleep(120);
    }

    function focusEditorAtEnd() {
        const editor = findEditor();
        if (!editor) return;

        editor.focus({ preventScroll: true });

        if (editor instanceof HTMLTextAreaElement) {
            const end = editor.value.length;
            editor.setSelectionRange(end, end);
            return;
        }

        const selection = window.getSelection();
        if (!selection) return;

        const range = document.createRange();
        range.selectNodeContents(editor);
        range.collapse(false);
        selection.removeAllRanges();
        selection.addRange(range);
    }

    function findEffortMenuDebug() {
        const preferred = document.querySelector(
            '[data-testid="composer-intelligence-picker-content"]'
        );

        if (preferred) return preferred;

        /*
         * data-testid変更時の保険。
         * role=menu のうち、応答性能らしいradio項目を2つ以上含むものだけ採用します。
         */
        for (const menu of document.querySelectorAll('[role="menu"]')) {
            const radios = [...menu.querySelectorAll('[role="menuitemradio"]')];
            const matched = radios.filter(item =>
                Boolean(inferLevelFromText(item.textContent))
            );

            if (matched.length >= 2) {
                return menu.querySelector('[role="group"]') || menu;
            }
        }

        return null;
    }

    async function waitForEffortMenu(timeoutMs = 15000) {
        return waitFor(() => findEffortMenuDebug(), timeoutMs, 50);
    }

    function debugElementSummary(element) {
        if (!element) return null;

        const rect = element.getBoundingClientRect();
        return {
            tag: element.tagName,
            id: element.id || null,
            text: normalizeText(element.textContent),
            ariaExpanded: element.getAttribute('aria-expanded'),
            ariaHaspopup: element.getAttribute('aria-haspopup'),
            dataState: element.getAttribute('data-state'),
            disabled: Boolean(element.disabled),
            connected: element.isConnected,
            visible: isVisible(element),
            rect: {
                left: Math.round(rect.left),
                top: Math.round(rect.top),
                width: Math.round(rect.width),
                height: Math.round(rect.height)
            },
            outerHTML: element.outerHTML.slice(0, 2000)
        };
    }

    async function waitBrieflyForEffortMenu(timeoutMs = 900) {
        const startedAt = Date.now();
        while (Date.now() - startedAt < timeoutMs) {
            const menu = findEffortMenuDebug();
            if (menu) return menu;
            await sleep(50);
        }
        return null;
    }

    function findDetailedSettingsItem(menu) {
        return [...menu.querySelectorAll(
            '[role="menuitem"], button, [tabindex="0"]'
        )].find(item =>
            isVisible(item) &&
            normalizeText(item.textContent).includes('詳細設定')
        ) || null;
    }

    async function ensureAdvancedView(menu) {
        setDebugStage('Work表示判定', summarizeMenu(menu));

        let reasoningItem = findReasoningSubmenuItem(menu);
        log('思考レベル候補（切替前）', debugElementSummary(reasoningItem));
        if (reasoningItem && isVisible(reasoningItem)) {
            log('すでに詳細設定表示です');
            return menu;
        }

        const detailedCandidates = [...menu.querySelectorAll(
            '[role="menuitem"], button, [tabindex="0"]'
        )].map(item => ({
            text: normalizeText(item.innerText || item.textContent),
            visible: isVisible(item),
            summary: debugElementSummary(item)
        }));
        log('詳細設定候補一覧', detailedCandidates);

        const detailedItem = findDetailedSettingsItem(menu);
        log('採用した詳細設定要素', debugElementSummary(detailedItem));
        if (!detailedItem) {
            warn('詳細設定要素が見つかりません。現在の親メニューを返します');
            dumpOpenMenus('詳細設定未検出時');
            return menu;
        }

        setDebugStage('詳細設定をクリック', debugElementSummary(detailedItem));
        dispatchPointerClick(detailedItem);
        await sleep(150);
        dumpOpenMenus('詳細設定クリック直後');

        reasoningItem = await waitFor(() => {
            const currentMenu = document.querySelector(
                '[data-testid="composer-intelligence-picker-content"]'
            ) || menu;
            const item = findReasoningSubmenuItem(currentMenu);
            return item && isVisible(item) ? item : null;
        }, 7000, 50);

        log('詳細設定への切替成功', debugElementSummary(reasoningItem));
        return document.querySelector(
            '[data-testid="composer-intelligence-picker-content"]'
        ) || menu;
    }

    function findReasoningSubmenuItem(menu) {
        return [...menu.querySelectorAll(
            '[role="menuitem"][aria-haspopup="menu"]'
        )].find(item =>
            normalizeText(item.textContent).includes('思考レベル')
        ) || null;
    }

    function findControlledMenu(item) {
        const id = item?.getAttribute('aria-controls');
        return id ? document.getElementById(id) : null;
    }

    function findReasoningOptionsMenu(item) {
        const controlled = findControlledMenu(item);
        if (
            controlled &&
            controlled.getAttribute('data-state') === 'open' &&
            controlled.querySelector('[role="menuitemradio"]')
        ) {
            return controlled;
        }

        return [...document.querySelectorAll('[role="menu"]')].find(menu =>
            menu !== item?.closest('[role="menu"]') &&
            isVisible(menu) &&
            menu.getAttribute('data-state') !== 'closed' &&
            menu.querySelector('[role="menuitemradio"]')
        ) || null;
    }

    function dispatchSubmenuOpen(item) {
        const rect = item.getBoundingClientRect();
        const common = {
            bubbles: true,
            cancelable: true,
            composed: true,
            clientX: rect.left + Math.max(4, rect.width - 12),
            clientY: rect.top + rect.height / 2,
            button: 0,
            buttons: 0
        };

        item.focus?.({ preventScroll: true });

        item.dispatchEvent(new PointerEvent('pointerover', {
            ...common, pointerId: 1, pointerType: 'mouse', isPrimary: true
        }));
        item.dispatchEvent(new PointerEvent('pointerenter', {
            ...common, pointerId: 1, pointerType: 'mouse', isPrimary: true
        }));
        item.dispatchEvent(new PointerEvent('pointermove', {
            ...common, pointerId: 1, pointerType: 'mouse', isPrimary: true
        }));
        item.dispatchEvent(new MouseEvent('mouseover', common));
        item.dispatchEvent(new MouseEvent('mouseenter', common));
        item.dispatchEvent(new MouseEvent('mousemove', common));

        dispatchPointerClick(item);

        item.dispatchEvent(new KeyboardEvent('keydown', {
            key: 'ArrowRight',
            code: 'ArrowRight',
            keyCode: 39,
            which: 39,
            bubbles: true,
            cancelable: true,
            composed: true
        }));
        item.dispatchEvent(new KeyboardEvent('keyup', {
            key: 'ArrowRight',
            code: 'ArrowRight',
            keyCode: 39,
            which: 39,
            bubbles: true,
            cancelable: true,
            composed: true
        }));
    }

    async function openReasoningSubmenu(parentMenu) {
        setDebugStage('思考レベルサブメニュー準備', summarizeMenu(parentMenu));
        parentMenu = await ensureAdvancedView(parentMenu);

        const item = findReasoningSubmenuItem(parentMenu);
        log('思考レベル要素', debugElementSummary(item));
        if (!item || !isVisible(item)) {
            warn('思考レベル要素を検出できません');
            dumpOpenMenus('思考レベル未検出時');
            return null;
        }

        let submenu = findReasoningOptionsMenu(item);
        if (submenu) {
            log('思考レベルサブメニューは既に開いています');
            return submenu;
        }

        setDebugStage('思考レベルを開く', debugElementSummary(item));
        dispatchSubmenuOpen(item);

        submenu = await waitFor(() => findReasoningOptionsMenu(item), 7000, 50);

        log('思考レベルサブメニュー検出成功', summarizeMenu(submenu));
        return submenu;
    }

    function parseEffortMenu(menu, optionsRoot = menu) {
        setDebugStage('レベル候補解析', { parent: summarizeMenu(menu), optionsRoot: summarizeMenu(optionsRoot) });
        const options = new Map();

        for (const item of optionsRoot.querySelectorAll(
            '[role="menuitemradio"]'
        )) {
            const labelElement = item.querySelector('.truncate');
            const label = normalizeText(
                labelElement?.textContent || item.textContent
            );
            const level = inferLevelFromText(label);

            if (!level) continue;

            options.set(level.key, {
                level,
                element: item,
                checked:
                    item.getAttribute('aria-checked') === 'true' ||
                    item.dataset.state === 'checked'
            });
        }

        const modelItem = [...menu.querySelectorAll(
            '[role="menuitem"]'
        )].find(item => {
            const text = normalizeText(item.textContent);
            return Boolean(
                text.startsWith('モデル') ||
                inferExplicitModelVersionFromText(text) ||
                /\bSol\b|\bWork\b|ワーク|GPT[-\s]*5/i.test(text)
            );
        });

        const modelText = normalizeText(modelItem?.textContent)
            .replace(/^モデル\s*/, '');

        log('解析したレベル候補', [...options.entries()].map(([key, value]) => ({ key, label: value.level.displayName, checked: value.checked, element: debugElementSummary(value.element) })));

        return {
            options,
            modelInfo: modelItem ? {
                text: modelText,
                version:
                    inferExplicitModelVersionFromText(modelText) ||
                    inferModelVersionFromText(modelText),
                element: modelItem
            } : null,
            workStyle: optionsRoot !== menu
        };
    }

    async function openEffortMenuAndInspect(currentButton) {
        if (!currentButton) {
            throw new Error(
                '現在の応答性能を表示するボタンが見つかりません'
            );
        }

        const existing = findEffortMenuDebug();
        if (existing) {
            log('メニューは既に開いています', debugElementSummary(existing));
            const reasoningSubmenu = await openReasoningSubmenu(existing);
            const inspection = parseEffortMenu(
                existing,
                reasoningSubmenu || existing
            );
            log('応答性能メニュー:', {
                levels: [...inspection.options.keys()],
                model: inspection.modelInfo?.text || null
            });
            return inspection;
        }

        log('応答性能ボタン詳細', debugElementSummary(currentButton));

        const rect = currentButton.getBoundingClientRect();
        const clientX = rect.left + rect.width / 2;
        const clientY = rect.top + rect.height / 2;
        const pointElement = document.elementFromPoint(clientX, clientY);
        log('ボタン中央のelementFromPoint', debugElementSummary(pointElement));

        const added = [];
        const mutationObserver = new MutationObserver(records => {
            for (const record of records) {
                for (const node of record.addedNodes) {
                    if (!(node instanceof Element)) continue;
                    const interesting =
                        node.matches?.('[role="menu"], [data-radix-popper-content-wrapper], [data-testid]') ||
                        node.querySelector?.('[role="menu"], [data-radix-popper-content-wrapper], [data-testid="composer-intelligence-picker-content"]');
                    if (interesting && added.length < 30) {
                        added.push({
                            tag: node.tagName,
                            id: node.id || null,
                            role: node.getAttribute('role'),
                            testid: node.getAttribute('data-testid'),
                            text: normalizeText(node.textContent).slice(0, 300),
                            html: node.outerHTML.slice(0, 1200)
                        });
                    }
                }
            }
        });
        mutationObserver.observe(document.body, { childList: true, subtree: true });

        const eventTypes = [
            'pointerdown', 'mousedown', 'pointerup', 'mouseup', 'click',
            'keydown', 'keyup'
        ];
        const eventLogger = event => {
            if (event.target === currentButton || currentButton.contains(event.target)) {
                log('ボタンイベント', {
                    type: event.type,
                    isTrusted: event.isTrusted,
                    key: event.key || null,
                    button: typeof event.button === 'number' ? event.button : null,
                    defaultPrevented: event.defaultPrevented,
                    ariaExpanded: currentButton.getAttribute('aria-expanded'),
                    dataState: currentButton.getAttribute('data-state')
                });
            }
        };
        for (const type of eventTypes) {
            document.addEventListener(type, eventLogger, true);
        }

        const common = {
            bubbles: true,
            cancelable: true,
            composed: true,
            clientX,
            clientY,
            button: 0,
            buttons: 1
        };

        const attempts = [
            {
                name: 'HTMLElement.prototype.click',
                run: () => {
                    currentButton.focus({ preventScroll: true });
                    HTMLElement.prototype.click.call(currentButton);
                }
            },
            {
                name: 'Pointer+Mouse sequence',
                run: () => {
                    currentButton.dispatchEvent(new PointerEvent('pointerdown', {
                        ...common,
                        pointerId: 1,
                        pointerType: 'mouse',
                        isPrimary: true
                    }));
                    currentButton.dispatchEvent(new MouseEvent('mousedown', common));
                    currentButton.dispatchEvent(new PointerEvent('pointerup', {
                        ...common,
                        buttons: 0,
                        pointerId: 1,
                        pointerType: 'mouse',
                        isPrimary: true
                    }));
                    currentButton.dispatchEvent(new MouseEvent('mouseup', {
                        ...common,
                        buttons: 0
                    }));
                    currentButton.dispatchEvent(new MouseEvent('click', {
                        ...common,
                        buttons: 0,
                        detail: 1
                    }));
                }
            },
            {
                name: 'Keyboard Enter',
                run: () => {
                    currentButton.focus({ preventScroll: true });
                    currentButton.dispatchEvent(new KeyboardEvent('keydown', {
                        key: 'Enter', code: 'Enter', bubbles: true, cancelable: true
                    }));
                    currentButton.dispatchEvent(new KeyboardEvent('keyup', {
                        key: 'Enter', code: 'Enter', bubbles: true, cancelable: true
                    }));
                }
            },
            {
                name: 'Keyboard Space',
                run: () => {
                    currentButton.focus({ preventScroll: true });
                    currentButton.dispatchEvent(new KeyboardEvent('keydown', {
                        key: ' ', code: 'Space', bubbles: true, cancelable: true
                    }));
                    currentButton.dispatchEvent(new KeyboardEvent('keyup', {
                        key: ' ', code: 'Space', bubbles: true, cancelable: true
                    }));
                }
            }
        ];

        let menu = null;
        try {
            for (const attempt of attempts) {
                log(`開く試行: ${attempt.name}`, {
                    beforeAriaExpanded: currentButton.getAttribute('aria-expanded'),
                    beforeDataState: currentButton.getAttribute('data-state')
                });

                attempt.run();
                await sleep(80);

                log(`試行直後: ${attempt.name}`, {
                    afterAriaExpanded: currentButton.getAttribute('aria-expanded'),
                    afterDataState: currentButton.getAttribute('data-state'),
                    activeElement: debugElementSummary(document.activeElement)
                });

                menu = await waitBrieflyForEffortMenu(1000);
                if (menu) {
                    log(`メニュー検出成功: ${attempt.name}`,
                        debugElementSummary(menu));
                    break;
                }
            }

            if (!menu) {
                log('自動操作では開けませんでした。15秒以内に「最速」を手動クリックしてください');
                menu = await waitForEffortMenu(15000);
                log('手動クリック後にメニューを検出', debugElementSummary(menu));
            }
        } finally {
            mutationObserver.disconnect();
            for (const type of eventTypes) {
                document.removeEventListener(type, eventLogger, true);
            }
            log('追加DOM候補', added);
            log('最終ボタン状態', debugElementSummary(currentButton));
            log('現在のRadix候補数', {
                wrappers: document.querySelectorAll('[data-radix-popper-content-wrapper]').length,
                menus: document.querySelectorAll('[role="menu"]').length,
                effortTestIds: document.querySelectorAll('[data-testid="composer-intelligence-picker-content"]').length
            });
        }

        const reasoningSubmenu = await openReasoningSubmenu(menu);
        const inspection = parseEffortMenu(
            menu,
            reasoningSubmenu || menu
        );
        log('応答性能メニュー:', {
            levels: [...inspection.options.keys()],
            model: inspection.modelInfo?.text || null
        });

        return inspection;
    }

    async function openEffortMenuAndGetOptions(currentButton) {
        const inspection =
            await openEffortMenuAndInspect(currentButton);

        return inspection.options;
    }

    function availableLevelKeysFrom(options, currentLevel) {
        const keys = new Set(options.keys());

        if (currentLevel) {
            keys.add(currentLevel.key);
        }

        return keys;
    }

    function chooseByPriority(priorityKeys, availableKeys) {
        for (const key of priorityKeys) {
            if (availableKeys.has(key)) {
                return getLevelByKey(key);
            }
        }

        return null;
    }

    async function resolveTargetEffort() {
        setDebugStage('現在環境を検出');
        const env = detectCurrentEnvironment();
        log('検出した環境', { button: debugElementSummary(env.button), buttonText: env.buttonText, currentLevel: env.currentLevel, modelVersion: env.modelVersion });

        if (!env.button) {
            return {
                ok: false,
                reason:
                    '現在の応答性能を取得できませんでした。\n\n' +
                    'GPT-5.5以上で、応答性能メニューが表示される状態で使ってください。'
            };
        }

        setDebugStage('応答性能メニューを解析');
        const inspection =
            await openEffortMenuAndInspect(env.button);

        /*
         * Workのデフォルト表示ではボタン文字だけで現在値を取れない場合があるため、
         * 開いた思考レベルメニューのaria-checkedも現在値の根拠にします。
         */
        const checkedOption = [...inspection.options.values()]
            .find(option => option.checked);
        const currentLevel = env.currentLevel || checkedOption?.level || null;

        if (!currentLevel) {
            closeEffortMenu();
            return {
                ok: false,
                reason:
                    '現在の応答性能を取得できませんでした。\n\n' +
                    '応答性能メニューを開ける状態で、もう一度試してください。'
            };
        }

        closeEffortMenu();
        await sleep(180);

        const options = inspection.options;
        const availableKeys =
            availableLevelKeysFrom(options, currentLevel);

        const modelVersion =
            inspection.modelInfo?.version ||
            env.modelVersion ||
            null;

        /*
         * 最大または非常に高いが実際に表示される環境をWork扱い。
         * Solというモデル名だけではWork判定しません。
         */
        const hasWorkLevels =
            availableKeys.has('max') ||
            availableKeys.has('extra-high');

        if (hasWorkLevels) {
            const target = chooseByPriority(
                TARGET_PRIORITY_WORK,
                availableKeys
            );

            if (!target) {
                return {
                    ok: false,
                    reason:
                        'Workの本気設定を確認できませんでした。\n\n' +
                        '「最大」「非常に高い」「高い」のいずれかが利用できる状態で使ってください。'
                };
            }

            return {
                ok: true,
                modeName: 'Work',
                previous: currentLevel,
                target,
                modelName:
                    inspection.modelInfo?.text ||
                    'GPT-5.5以上'
            };
        }

        /*
         * 通常版では、現在が最速または中程度でも
         * 「高い」が実在すれば高いへ変更して送信します。
         */
        if (availableKeys.has('high')) {
            if (
                modelVersion &&
                !modelVersion.isAtLeast55
            ) {
                return {
                    ok: false,
                    reason:
                        'この機能はGPT-5.5以上で利用できます。\n\n' +
                        '現在のモデルはGPT-5.5未満です。GPT-5.5以上へ切り替えてから使ってください。'
                };
            }

            return {
                ok: true,
                modeName: '通常版',
                previous: currentLevel,
                target: getLevelByKey('high'),
                modelName:
                    inspection.modelInfo?.text ||
                    modelVersion?.raw ||
                    'GPT-5.5以上'
            };
        }

        if (
            modelVersion &&
            !modelVersion.isAtLeast55
        ) {
            return {
                ok: false,
                reason:
                    'この機能はGPT-5.5以上で利用できます。\n\n' +
                    `現在のモデルは${inspection.modelInfo?.text || modelVersion.raw}です。` +
                    'GPT-5.5以上へ切り替えてから使ってください。'
            };
        }

        return {
            ok: false,
            reason:
                '推論レベル「高い」を確認できませんでした。\n\n' +
                'GPT-5.5以上へ切り替え、応答性能メニューに「高い」が表示される状態で使ってください。'
        };
    }


    async function resolveAttachmentEffort() {
        setDebugStage('添付送信の環境を検出');
        const env = detectCurrentEnvironment();

        if (!env.button) {
            throw new Error('現在の応答性能を取得できませんでした');
        }

        const inspection = await openEffortMenuAndInspect(env.button);
        const checkedOption = [...inspection.options.values()]
            .find(option => option.checked);
        const currentLevel = env.currentLevel || checkedOption?.level || null;

        if (!currentLevel) {
            closeEffortMenu();
            throw new Error('現在の応答性能を取得できませんでした');
        }

        closeEffortMenu();
        await sleep(180);

        const availableKeys = availableLevelKeysFrom(
            inspection.options,
            currentLevel
        );
        const isWork =
            availableKeys.has('max') ||
            availableKeys.has('extra-high');
        const target = chooseByPriority(
            isWork ? ATTACHMENT_PRIORITY_WORK : ATTACHMENT_PRIORITY_NORMAL,
            availableKeys
        );

        if (!target) {
            throw new Error(
                isWork
                    ? 'Workの「軽量」を確認できませんでした'
                    : '通常Chatの「最速」を確認できませんでした'
            );
        }

        return {
            previous: currentLevel,
            target,
            modeName: isWork ? 'Work' : '通常版'
        };
    }

    async function submitAttachmentOptimized(sendButton) {
        if (attachmentSubmitInProgress) return;

        const editor = findEditor();
        if (!editor || !composerHasAttachment(editor)) return;
        if (!sendButton || sendButton.disabled) return;

        attachmentSubmitInProgress = true;
        let previousEffort = null;
        let shouldRestore = false;

        try {
            const resolved = await resolveAttachmentEffort();
            previousEffort = resolved.previous;
            shouldRestore =
                previousEffort.key !== resolved.target.key;

            if (shouldRestore) {
                await setEffort(resolved.target);
            }

            const refreshedSendButton = findSendButton(findEditor());
            if (!refreshedSendButton || refreshedSendButton.disabled) {
                throw new Error('送信ボタンが見つからないか、現在送信できません');
            }

            const assistantCountBeforeSend = countAssistantMessages();
            await sleep(120);
            refreshedSendButton.click();

            await waitForGenerationToFinish(assistantCountBeforeSend);

            if (shouldRestore) {
                await setEffort(previousEffort);
            }
        } catch (error) {
            // console.error(`[${SCRIPT_NAME} v${SCRIPT_VERSION}] 添付送信エラー`, error);

            try {
                const current = detectCurrentEffortLevel();
                if (
                    previousEffort &&
                    current &&
                    current.key !== previousEffort.key
                ) {
                    await setEffort(previousEffort);
                }
            } catch (_) {
                // 復元失敗時は画面上の設定を手動確認してください。
            }

            alert(
                '添付ファイル用の自動切替に失敗しました。\n\n' +
                `${error.message}\n\n` +
                '推論レベルを確認して、もう一度送信してください。'
            );
        } finally {
            await closeEffortMenuFully();
            attachmentSubmitInProgress = false;
            setTimeout(focusEditorAtEnd, 0);
        }
    }

    function installAttachmentSubmitInterceptor() {
        document.addEventListener('click', event => {
            if (attachmentSubmitInProgress) return;

            const clickedButton = event.target?.closest?.('button');
            const editor = findEditor();
            const sendButton = findSendButton(editor);

            if (
                !clickedButton ||
                clickedButton !== sendButton ||
                !composerHasAttachment(editor)
            ) {
                return;
            }

            event.preventDefault();
            event.stopImmediatePropagation();
            submitAttachmentOptimized(sendButton);
        }, true);

        document.addEventListener('keydown', event => {
            if (
                attachmentSubmitInProgress ||
                event.key !== 'Enter' ||
                event.shiftKey ||
                event.ctrlKey ||
                event.altKey ||
                event.metaKey ||
                event.isComposing
            ) {
                return;
            }

            const editor = findEditor();
            if (
                !editor ||
                !editor.contains(event.target) ||
                !composerHasAttachment(editor)
            ) {
                return;
            }

            const sendButton = findSendButton(editor);
            if (!sendButton || sendButton.disabled) return;

            event.preventDefault();
            event.stopImmediatePropagation();
            submitAttachmentOptimized(sendButton);
        }, true);
    }

    async function setEffort(level) {
        setDebugStage(`推論レベル変更開始: ${level?.displayName || '未指定'}`);
        if (!level) {
            throw new Error('変更先の推論レベルが指定されていません');
        }

        const env = detectCurrentEnvironment();

        if (!env.button) {
            throw new Error(
                '推論レベルを表示するボタンが見つかりません'
            );
        }

        if (env.currentLevel?.key === level.key) {
            log(`推論レベルはすでに「${level.displayName}」です`);
            return;
        }

        const options = await openEffortMenuAndGetOptions(env.button);
        log('変更時の候補一覧', [...options.keys()]);
        const target = options.get(level.key);

        if (!target) {
            closeEffortMenu();

            throw new Error(
                `推論レベル「${level.displayName}」が見つかりません`
            );
        }

        log('クリックするレベル要素', debugElementSummary(target.element));
        dispatchPointerClick(target.element);
        await sleep(150);
        dumpOpenMenus('レベルクリック直後');

        await waitFor(() => {
            const detected = detectCurrentEffortLevel();
            return detected?.key === level.key ? detected : null;
        }, 7000, 150);

        log(`推論レベルを「${level.displayName}」へ変更しました`);
        await closeEffortMenuFully();
    }

    async function waitForGenerationToFinish(assistantCountBeforeSend) {
        await waitFor(() => {
            return (
                findStopButton() ||
                countAssistantMessages() > assistantCountBeforeSend
            );
        }, 30000, 150);

        if (findStopButton()) {
            await waitFor(
                () => !findStopButton(),
                60 * 60 * 1000,
                500
            );
        }

        /*
         * 短い回答で停止ボタンが見えない場合に備え、
         * 最新回答が2秒間変化しないことを確認します。
         */
        let stableSince = Date.now();
        let previousText = getLatestAssistantText();

        while (Date.now() - stableSince < 2000) {
            await sleep(400);

            const currentText = getLatestAssistantText();

            if (currentText !== previousText) {
                previousText = currentText;
                stableSince = Date.now();
            }
        }
    }

    function setButtonAppearance(button, state = 'idle', level = null) {
        const targetName = level?.displayName || '指定設定';

        const states = {
            idle: {
                text: '🔥 本気で聞く',
                title:
                    '最大・非常に高いがある環境では最大優先、通常GPT-5.5以上では高い優先で送信し、' +
                    '送信開始後に元の設定へ戻します'
            },
            checking: {
                text: '🔎 設定確認中…',
                title: '現在のモデルと推論レベルを確認しています'
            },
            changing: {
                text: `🔥 ${targetName}へ変更中…`,
                title: `推論レベルを${targetName}へ変更しています`
            },
            sending: {
                text: '🔥 送信！',
                title: `${targetName}で送信します`
            },
            waiting: {
                text: '🔥 回答待ち…',
                title: '生成開始を確認後、元の推論レベルへ戻します'
            },
            restoring: {
                text: '↩ 元の設定へ戻す…',
                title: '押す前の推論レベルへ戻しています'
            }
        };

        const selected = states[state] || states.idle;
        button.textContent = selected.text;
        button.title = selected.title;
    }

    async function seriousSubmit(button) {
        setDebugStage('🔥ボタン押下');
        dumpOpenMenus('処理開始時');
        const editor = findEditor();

        if (!editor || !editorHasText(editor)) {
            alert('質問が入力されていません。');
            focusEditorAtEnd();
            return;
        }

        /* OK前はChatGPT側のメニューを一切開かない。 */
        const confirmed = window.confirm(
            '🔥 本気モードで送信しますか？\n\n' +
            'OKを押すと応答性能を確認し、利用可能な最上位設定で送信後、元へ戻します。'
        );

        if (!confirmed) {
            closeEffortMenu();
            setTimeout(focusEditorAtEnd, 0);
            return;
        }

        button.disabled = true;

        let previousEffort = null;
        let targetEffort = null;
        let shouldRestore = false;

        try {
            setButtonAppearance(button, 'checking');

            const resolved = await resolveTargetEffort();

            if (!resolved.ok) {
                alert(resolved.reason);
                return;
            }

            previousEffort = resolved.previous;
            targetEffort = resolved.target;
            shouldRestore =
                previousEffort &&
                targetEffort &&
                previousEffort.key !== targetEffort.key;

            if (shouldRestore) {
                setButtonAppearance(
                    button,
                    'changing',
                    targetEffort
                );

                await setEffort(targetEffort);
            }

            const refreshedEditor = findEditor();
            const sendButton = findSendButton(refreshedEditor);

            if (!sendButton || sendButton.disabled) {
                throw new Error(
                    '送信ボタンが見つからないか、現在送信できません'
                );
            }

            const assistantCountBeforeSend = countAssistantMessages();

            setButtonAppearance(
                button,
                'sending',
                targetEffort
            );

            await sleep(200);
            sendButton.click();

            setButtonAppearance(
                button,
                'waiting',
                targetEffort
            );

            /*
             * 通常Chat・Workともに回答完了を待ち、
             * 最新回答が2秒間変化しないことを確認してから元へ戻す。
             */
            await waitForGenerationToFinish(assistantCountBeforeSend);

            if (shouldRestore) {
                setButtonAppearance(button, 'restoring');
                await setEffort(previousEffort);
            }

        } catch (error) {
            dumpOpenMenus('エラー発生時');
            // console.error(`[${SCRIPT_NAME} v${SCRIPT_VERSION}]`, error);

            let restoreMessage = '';

            try {
                const current = detectCurrentEffortLevel();

                if (
                    previousEffort &&
                    current &&
                    current.key !== previousEffort.key
                ) {
                    setButtonAppearance(button, 'restoring');
                    await setEffort(previousEffort);

                    restoreMessage =
                        `\n\n元の設定「${previousEffort.displayName}」へ戻しました。`;
                }
            } catch (restoreError) {
                warn('元の設定へ戻せませんでした', restoreError);

                restoreMessage =
                    `\n\n自動で元の設定へ戻せませんでした。` +
                    `\n手動で「${previousEffort?.displayName || '元の設定'}」` +
                    'へ戻してください。';
            }

            alert(
                '本気で聞くボタンの処理に失敗しました。\n\n' +
                `停止段階：${DEBUG_STAGE}\n\n` +
                `${error.message}${restoreMessage}\n\n` +
                'ChatGPT側の画面構造が変更された可能性があります。'
            );

        } finally {
            await closeEffortMenuFully();
            button.disabled = false;
            setButtonAppearance(button, 'idle');
            setTimeout(focusEditorAtEnd, 0);
        }
    }

    function styleButton(button) {
        Object.assign(button.style, {
            height: '32px',
            padding: '0 11px',
            marginRight: '7px',
            border: '1px solid rgba(255, 120, 50, 0.72)',
            borderRadius: '16px',
            background: 'rgba(255, 92, 40, 0.13)',
            color: 'inherit',
            fontSize: '12px',
            fontWeight: '650',
            lineHeight: '1',
            whiteSpace: 'nowrap',
            cursor: 'pointer',
            transition:
                'background 0.15s ease, opacity 0.15s ease, ' +
                'transform 0.15s ease'
        });

        button.addEventListener('mouseenter', () => {
            if (!button.disabled) {
                button.style.background =
                    'rgba(255, 92, 40, 0.25)';
                button.style.transform = 'translateY(-1px)';
            }
        });

        button.addEventListener('mouseleave', () => {
            button.style.background =
                'rgba(255, 92, 40, 0.13)';
            button.style.transform = 'translateY(0)';
        });
    }

    function installButton() {
        if (document.getElementById(BUTTON_ID)) {
            return;
        }

        const editor = findEditor();
        const sendButton = findSendButton(editor);

        if (!editor || !sendButton || !sendButton.parentElement) {
            return;
        }

        const button = document.createElement('button');

        button.id = BUTTON_ID;
        button.type = 'button';

        setButtonAppearance(button, 'idle');
        styleButton(button);

        button.addEventListener('click', () => {
            seriousSubmit(button);
        });

        sendButton.parentElement.insertBefore(
            button,
            sendButton
        );

        log('「🔥 本気で聞く」ボタンを追加しました');
    }

    /*
     * 初期hydration中にはDOMを書き換えない。
     * DOM変化のたびに即座に全探索せず、アイドル時にまとめて1回だけ確認する。
     */
    let installTimer = null;
    let installQueued = false;

    function scheduleInstall(delayMs = 700) {
        clearTimeout(installTimer);
        installTimer = setTimeout(() => {
            if (installQueued) return;
            installQueued = true;

            const run = () => {
                installQueued = false;
                installButton();
            };

            if ('requestIdleCallback' in window) {
                window.requestIdleCallback(run, { timeout: 1200 });
            } else {
                setTimeout(run, 0);
            }
        }, delayMs);
    }

    const observer = new MutationObserver(records => {
        const relevant = records.some(record =>
            [...record.addedNodes, ...record.removedNodes]
                .some(node => node instanceof Element)
        );
        if (relevant) scheduleInstall(700);
    });

    observer.observe(document.body, {
        childList: true,
        subtree: true
    });

    const startAfterHydration = () => scheduleInstall(2200);
    if (document.readyState === 'complete') {
        startAfterHydration();
    } else {
        window.addEventListener('load', startAfterHydration, { once: true });
    }
    installAttachmentSubmitInterceptor();

})();
