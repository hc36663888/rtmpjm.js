// ==UserScript==
// @name         RTMPS 推流码捕获器（卡密验证 + 服务器解析）
// @namespace    http://tampermonkey.net/
// @version      3.0
// @description  捕获 RTMPS 推流地址，将原始URL发送到服务器解析；卡密验证由服务器控制。
// @match        *://*/*
// @grant        none

// ====== GitHub Auto Update（自动更新） ======
// @updateURL   https://raw.githubusercontent.com/hc36663888/rtmpjm.js/main/rtmpjm.js
// @downloadURL https://raw.githubusercontent.com/hc36663888/rtmpjm.js/main/rtmpjm.js
// @supportURL  https://github.com/hc36663888/rtmpjm.js/issues
// ============================================

// ==/UserScript==

(async function () {

    const API_BASE = "https://hc11.pw58.cc/rtmp";
    const LICENSE_API = API_BASE + "/rt_check_license.php";
    const DECODE_API  = API_BASE + "/rt_decode_rtmps.php";
    const STORAGE_KEY = "rtmps_license_key";

    function getHWID() {
        const raw = navigator.userAgent + "|" + screen.width + "x" + screen.height;
        return btoa(raw);
    }

    async function apiGet(url) {
        const res = await fetch(url, { cache: "no-store" });
        return res.json();
    }

    async function apiPost(url, data) {
        const res = await fetch(url, {
            method: "POST",
            headers: {"Content-Type":"application/json"},
            body: JSON.stringify(data)
        });
        return res.json();
    }

    async function verifyLicense(key) {
        const hwid = getHWID();
        const url = `${LICENSE_API}?key=${encodeURIComponent(key)}&hwid=${encodeURIComponent(hwid)}`;
        const data = await apiGet(url);
        return data.valid === true;
    }

    async function checkLicense() {
        let saved = localStorage.getItem(STORAGE_KEY);

        if (saved) {
            if (await verifyLicense(saved)) return true;
            else localStorage.removeItem(STORAGE_KEY);
        }

        for (let i = 0; i < 3; i++) {
            const input = prompt("请输入卡密激活使用：");
            if (!input) continue;

            if (await verifyLicense(input)) {
                localStorage.setItem(STORAGE_KEY, input);
                alert("卡密验证成功！");
                return true;
            } else {
                alert("卡密错误、到期或被封禁，请重新输入。");
            }
        }

        alert("验证失败，脚本停止运行。");
        return false;
    }

    const ok = await checkLicense();
    if (!ok) return;

    async function sendRawToServer(rawUrl) {
        const hwid = getHWID();
        const key  = localStorage.getItem(STORAGE_KEY) || "";

        const data = await apiPost(DECODE_API, {
            raw: rawUrl,
            hwid: hwid,
            key: key,
            page: location.href
        });

        return data;
    }

    const originalFetch = window.fetch;
    window.fetch = async function (...args) {
        const response = await originalFetch.apply(this, args);
        try {
            const text = await response.clone().text();
            extract(text);
        } catch {}
        return response;
    };

    const originalOpen = XMLHttpRequest.prototype.open;
    XMLHttpRequest.prototype.open = function (...args) {
        this.addEventListener("load", function () {
            try {
                extract(this.responseText);
            } catch {}
        });
        return originalOpen.apply(this, args);
    };

    function extract(text) {
        const reg = /(rtmps:\/\/[^\s"'\\]+)/g;
        const matches = text.match(reg);
        if (!matches) return;

        for (const url of matches) {
            sendRawToServer(url).then(res => {
                if (!res || !res.ok) return;
                show(res.server, res.streamKey);
            });
        }
    }

    function show(server, key) {
        const old = document.getElementById("rtmps-box");
        if (old) old.remove();

        const div = document.createElement("div");
        div.id = "rtmps-box";
        div.style = `
            position:fixed; bottom:10px; right:10px; z-index:999999;
            background:#222; color:#fff; padding:15px; border-radius:8px;
            width:360px; max-height:160px; overflow:auto; font-family:monospace;
        `;
        div.innerHTML = `
            <b>捕获 RTMPS 推流信息</b><br><br>
            <b>服务器：</b><br>${server}<br><br>
            <b>StreamKey：</b><br>${key}
        `;
        document.body.appendChild(div);
        setTimeout(()=>div.remove(),60000);
    }

})();
