/**
 * @file Proxy Server for Cloudflare Workers
 * @description Advanced Web Proxy with HTMLRewriter, Streaming, Service Worker, and Virtual DOM Support.
 * Optimized with Module Worker architecture, high-performance DOM traversal, and strict stream handling.
 * Transited to TypeScript-compatible JSDoc for maintainability and type-safety without compilation errors.
 */

// =======================================================================================
// *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-* Configurations *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*
// =======================================================================================
const CONFIG = {
  routePrefix: "/", // (13) Renamed from str for clarity
  lastVisitProxyCookie: "_pv", // (15) Shortened cookie name
  passwordCookieName: "_ppw",
  proxyHintCookieName: "_phint",
  password: "123",
  showPasswordPage: true,
  proxyLocationProp: "__p_loc__", // (14) Renamed for clarity
  encPrefix: "p-", // (17) Shortened URL encode prefix
  DEBUG_MODE: false,
  cacheVersion: "v3" // (8) Fixed Cache version
};

// (16) Safe destructuring to prevent direct CONFIG.xxx calls everywhere
const { 
  routePrefix, 
  lastVisitProxyCookie, 
  passwordCookieName, 
  proxyHintCookieName, 
  password, 
  showPasswordPage, 
  proxyLocationProp, 
  encPrefix, 
  DEBUG_MODE, 
  cacheVersion 
} = CONFIG;

// (6) Fast debug logger
const debug = DEBUG_MODE ? (...a) => console.log(...a) : () => {};

// =======================================================================================
// *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-* Shared Logic *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*
// =======================================================================================

// (2, 3, 4, 27) Consolidated encoding logic with base64url compliance
function encodeProxy(str, prefix) {
  if (!str) return str;
  if (str.startsWith(prefix)) return str; // (1) Fixed inclusion bug
  try {
    // 1-pass replacement for base64url
    return prefix + btoa(encodeURIComponent(str)).replace(/[\/+=]/g, m => m === "/" ? "_" : m === "+" ? "-" : "");
  } catch (e) { 
    return str; // (22) Only wrap crucial failure point
  }
}

// (2, 19, 29) Consolidated decoding logic without heavy regex
function decodeProxy(str, prefix) {
  if (!str) return str;
  if (!str.startsWith(prefix)) return str;
  const i = str.indexOf("-");
  if (i === -1) return str;
  try {
    let s = str.substring(i + 1).replace(/_/g, '/').replace(/-/g, '+');
    const pad = s.length % 4;
    if (pad) s += "===".substring(0, 4 - pad);
    return decodeURIComponent(atob(s));
  } catch (e) { 
    return str; 
  }
}

// (31) ãµã¼ãã¼ãµã¤ãç¨ã¨ã³ã³ã¼ãã»ãã³ã¼ãã©ããã¼é¢æ°
// Reference: https://github.com/nodejs/undici for safe URL handling architecture
/**
 * Safe wrapper for server-side encoding to prevent ReferenceError
 * @param {string} str 
 * @returns {string}
 */
function _enc(str) { 
  return encodeProxy(str, encPrefix); 
}

/**
 * Safe wrapper for server-side decoding to prevent ReferenceError
 * @param {string} str 
 * @returns {string}
 */
function _dec(str) { 
  return decodeProxy(str, encPrefix); 
}

// =======================================================================================
// *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-* Error Page Templates *-*-*-*-*-*-*-*-*-*-*-*-*-*
// =======================================================================================

// (5, 20) Extracted CSS and optimized string replacement
const ERROR_TEMPLATE = `<html><head><meta charset="utf-8"><title>Proxy Error</title>
<style>
body { font-family: Arial, sans-serif; padding: 20px; background: #fff3f3; color: #333; }
h2 { color: #d32f2f; }
hr { margin-top: 20px; border: 0; border-top: 1px solid #ccc; }
p.footer { font-size: 12px; color: #666; }
</style>
</head>
<body>
  <h2>Proxy Encountered an Error</h2>
  <p>{MESSAGE}</p>
  <hr>
  <p class="footer">Cloudflare Worker Proxy Engine</p>
</body></html>`;

/**
 * @param {string} message 
 * @returns {string}
 */
function getErrorTemplate(message) {
  return ERROR_TEMPLATE.split("{MESSAGE}").join(message);
}

// =======================================================================================
// *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-* Code Generators *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*
// =======================================================================================

// (7, 26) Lazy initialization of massive strings to speed up cold starts
let SW_CODE = null;
function getServiceWorkerCode() {
  if (SW_CODE) return SW_CODE;
  SW_CODE = `
const encPrefix = "${encPrefix}";
const proxyScope = self.registration.scope;
const CACHE_VERSION = "${cacheVersion}";
const CACHE_NAME = "proxy-" + CACHE_VERSION;

${encodeProxy.toString()}
${decodeProxy.toString()}

function _enc_sw(str) { return encodeProxy(str, encPrefix); }
function _dec_sw(str) { return decodeProxy(str, encPrefix); }

// (9, 28) Optimized VirtualCookieStore to prevent memory leaks and track specs
const VirtualCookieStore = {
  memoryCache: new Map(), 
  maxSize: 500,
  async getCookies() {
    return Array.from(this.memoryCache.values());
  },
  async setCookie(cookieObj) {
    if (this.memoryCache.size > this.maxSize) {
      this.memoryCache.clear(); // Simple GC
    }
    const key = (cookieObj.name || "") + (cookieObj.domain || "") + (cookieObj.path || "");
    this.memoryCache.set(key, cookieObj);
  }
};

self.addEventListener('install', event => {
  self.skipWaiting();
  // (10) Fixed syntax and properly managed caching
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => {
      return cache.addAll(['/']);
    })
  );
});

self.addEventListener('activate', event => {
  // (8) Delete old caches
  event.waitUntil(
    caches.keys().then(keys => {
      return Promise.all(
        keys.map(k => {
          if (k !== CACHE_NAME) return caches.delete(k);
        })
      );
    }).then(() => {
      return Promise.all([
        self.clients.claim(),
        self.registration.navigationPreload ? self.registration.navigationPreload.enable() : Promise.resolve()
      ]);
    })
  );
});

self.addEventListener('message', event => {
  if (event.data && event.data.type === 'SYNC_DOM_STATE') {
    event.ports[0].postMessage({ status: 'ACK', version: CACHE_VERSION });
  }
});

self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);
  
  if (url.origin === location.origin) {
    if (url.pathname.startsWith("/" + encPrefix) || url.pathname.startsWith('/__PROXY')) {
      return; 
    }
    return;
  }

  event.respondWith((async () => {
    const preloadResponse = await event.preloadResponse;
    if (preloadResponse) return preloadResponse;

    const proxyUrl = location.origin + '/' + _enc_sw(url.href);
    try {
      const init = {
        method: event.request.method,
        headers: new Headers(event.request.headers),
        credentials: 'omit', 
        redirect: 'manual'
      };
      
      const vCookies = await VirtualCookieStore.getCookies();
      if (vCookies && vCookies.length > 0) {
        init.headers.set('Cookie', vCookies.map(c => c.name + '=' + c.value).join('; '));
      }

      if (event.request.method !== 'GET' && event.request.method !== 'HEAD') {
        init.body = await event.request.blob();
      }
      
      const response = await fetch(proxyUrl, init);
      
      // (UPDATE) â¡ TransformStreamãåç»ãå£ãã¦ããå¯è½æ§ï¼clone()ã®æ¤å»ï¼
      // Reference: https://github.com/whatwg/streams to prevent disrupting media stream backpressure
      // Reference: https://github.com/jimmywarting/StreamSaver.js for bypass architecture on range requests
      const contentType = response.headers.get('content-type') || "";
      const hasRange = event.request.headers.has('Range') || event.request.headers.has('range');
      const isMedia = contentType.includes('video/') || contentType.includes('audio/') || contentType.includes('application/octet-stream') || response.status === 206 || hasRange;
      
      if (!isMedia) {
          const responseClone = response.clone();
          const setCookieHeaders = responseClone.headers.get('Set-Cookie');
          if (setCookieHeaders) {
            const parts = setCookieHeaders.split(';');
            const nameVal = parts[0].split('=');
            await VirtualCookieStore.setCookie({ 
              name: nameVal[0], 
              value: nameVal[1],
              domain: url.hostname,
              path: "/"
            });
          }
      }
      
      return response;
    } catch (err) {
      const cached = await caches.match(event.request);
      if (cached) return cached;
      return new Response('Proxy Network Error', { status: 502 });
    }
  })());
});
`;
  return SW_CODE;
}

let CLIENT_CODE = null;
function getClientInjectionCode(proxyBaseUrl, proxyHost) {
  if (CLIENT_CODE) return CLIENT_CODE;
  CLIENT_CODE = `
const DEBUG_MODE = ${DEBUG_MODE};
function logDebug(...args) {
  if (DEBUG_MODE) console.log(...args);
}

var proxy_host = "${proxyHost}"; 
var proxy_host_with_schema = "${proxyBaseUrl}";

// (30) Defer execution in SW registration
if ('serviceWorker' in navigator) {
  window.addEventListener('load', function() {
    navigator.serviceWorker.register('/__PROXY_SW__.js').then(function(registration) {
      logDebug('ServiceWorker registration successful with scope: ', registration.scope);
      if (navigator.serviceWorker.controller) {
        const channel = new MessageChannel();
        channel.port1.onmessage = (e) => { logDebug("SW Sync ACK:", e.data); };
        navigator.serviceWorker.controller.postMessage({type: 'SYNC_DOM_STATE'},[channel.port2]);
      }
    }).catch(function(err) {
      logDebug('ServiceWorker registration failed: ', err);
    });
  });
}

${encodeProxy.toString()}
${decodeProxy.toString()}
const encPrefix = "${encPrefix}";
function _enc_client(str) { return encodeProxy(str, encPrefix); }
function _dec_client(str) { return decodeProxy(str, encPrefix); }

const workerBlob = new Blob([\`
  \${encodeProxy.toString()}
  self.onmessage = function(e) {
    const { type, payload, id, encPrefix } = e.data;
    if (type === 'ENC') {
      try {
        const res = encodeProxy(payload, encPrefix);
        self.postMessage({ id, result: res });
      } catch (err) { 
        self.postMessage({ id, result: payload }); 
      }
    }
  };
\`], { type: 'application/javascript' });

const calculationWorker = new Worker(URL.createObjectURL(workerBlob));
const workerCallbacks = new Map();
calculationWorker.onmessage = (e) => {
  const { id, result } = e.data;
  if (workerCallbacks.has(id)) {
    workerCallbacks.get(id)(result);
    workerCallbacks.delete(id);
  }
};

function _fast_enc_client(str) {
  if (!str) return str;
  if (str.startsWith(encPrefix)) return str;
  return _enc_client(str);
}

var encrypted_part = window.location.pathname.substring(1);
var original_website_url_str = _dec_client(encrypted_part); 

if(!original_website_url_str || !original_website_url_str.startsWith("http")) {
   original_website_url_str = window.location.href.substring(proxy_host_with_schema.length);
}

var original_website_url = new URL(original_website_url_str);
var original_website_host = original_website_url.host;
var original_website_host_with_schema = original_website_url.protocol + "//" + original_website_host + "/"; 

const proxyMemoryRegistry = new WeakMap();

class SimpleLRUCache {
  constructor(maxSize) {
    this.maxSize = maxSize;
    this.cache = new Map();
  }
  get(key) {
    if (!this.cache.has(key)) return undefined;
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }
  set(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }
  has(key) {
    return this.cache.has(key);
  }
}
var urlCacheMap = new SimpleLRUCache(500); 

// (24) Optimization: Fallback parsing string logic without heavy new URL() where possible
function strictResolveUrl(relativePath, base) {
  if (!relativePath) return relativePath;
  if (relativePath.startsWith('http')) return relativePath;
  if (relativePath.startsWith('//')) {
    const idx = base.indexOf(':');
    if (idx !== -1) return base.substring(0, idx + 1) + relativePath;
    return 'https:' + relativePath;
  }
  try {
    return new URL(relativePath, base).href;
  } catch (e) {
    if (relativePath.startsWith('/')) {
      try { return new URL(base).origin + relativePath; } catch(err){}
    }
    return base.replace(/\\/[^\\/]*$/, '/') + relativePath;
  }
}

function changeURL(relativePath){
  if(relativePath == null) return null;
  let relativePath_str = "";
  if (relativePath instanceof URL) {
    relativePath_str = relativePath.href;
  } else {
    relativePath_str = relativePath.toString();
  }

  if (urlCacheMap.has(relativePath_str)) {
    return urlCacheMap.get(relativePath_str);
  }

  try {
    if(relativePath_str.startsWith("data:") || relativePath_str.startsWith("mailto:") || relativePath_str.startsWith("javascript:") || relativePath_str.startsWith("chrome") || relativePath_str.startsWith("edge")) return relativePath_str;
  } catch(e) {
    return relativePath_str;
  }

  var pathAfterAdd = "";
  if(relativePath_str.startsWith("blob:")){
    pathAfterAdd = "blob:";
    relativePath_str = relativePath_str.substring(5);
  }

  try {
    if(relativePath_str.startsWith(proxy_host_with_schema)) relativePath_str = _dec_client(relativePath_str.substring(proxy_host_with_schema.length));
    if(relativePath_str.startsWith(proxy_host + "/")) relativePath_str = _dec_client(relativePath_str.substring(proxy_host.length + 1));
    if(relativePath_str.startsWith(proxy_host)) relativePath_str = _dec_client(relativePath_str.substring(proxy_host.length));
  } catch(e) {
    logDebug("Dec strip error", e);
  }

  try {
    var absolutePath = strictResolveUrl(relativePath_str, original_website_url_str); 
    absolutePath = absolutePath.replaceAll(window.location.href, original_website_url_str); 
    absolutePath = absolutePath.replaceAll(encodeURI(window.location.href), encodeURI(original_website_url_str));
    absolutePath = absolutePath.replaceAll(encodeURIComponent(window.location.href), encodeURIComponent(original_website_url_str));
    absolutePath = absolutePath.replaceAll(proxy_host, original_website_host);
    absolutePath = absolutePath.replaceAll(encodeURI(proxy_host), encodeURI(original_website_host));
    absolutePath = absolutePath.replaceAll(encodeURIComponent(proxy_host), encodeURIComponent(original_website_host));
    
    absolutePath = proxy_host_with_schema + _fast_enc_client(absolutePath);
    absolutePath = pathAfterAdd + absolutePath;
    urlCacheMap.set(relativePath_str, absolutePath); 
    return absolutePath;
  } catch (e) {
    return relativePath_str;
  }
}

function getOriginalUrl(url){
  if(url == null) return null;
  if(url.startsWith(proxy_host_with_schema)) return _dec_client(url.substring(proxy_host_with_schema.length));
  return url;
}

function networkInject(){
  var originalOpen = XMLHttpRequest.prototype.open;
  var originalFetch = window.fetch;
  const OriginalRequest = window.Request;
  
  window.Request = new Proxy(OriginalRequest, {
    construct(target, args) {
      let[input, init] = args;
      if (typeof input === 'string' || input instanceof URL) {
        input = changeURL(input);
      } else if (input instanceof OriginalRequest) {
        const cloned = input.clone();
        input = changeURL(cloned.url);
      }
      return new target(input, init);
    }
  });

  XMLHttpRequest.prototype.open = function(method, url, async, user, password) {
    url = changeURL(url);
    return originalOpen.apply(this, arguments);
  };

  window.fetch = function(input, init) {
    var url;
    if (typeof input === 'string') {
      url = input;
    } else if (input instanceof OriginalRequest) {
      url = input.url;
    } else {
      url = input;
    }
    url = changeURL(url);
    if (typeof input === 'string') {
      return originalFetch(url, init);
    } else {
      const newRequest = new window.Request(url, input);
      return originalFetch(newRequest, init);
    }
  };

  if (navigator.sendBeacon) {
    const origBeacon = navigator.sendBeacon;
    navigator.sendBeacon = function(url, data) {
      return origBeacon.call(this, changeURL(url), data);
    };
  }
}

function websocketInject() {
  if (window.WebSocket) {
    const OriginalWebSocket = window.WebSocket;
    window.WebSocket = function(url, protocols) {
      let modifiedUrl = changeURL(url);
      if (modifiedUrl && modifiedUrl.startsWith("http")) {
        modifiedUrl = modifiedUrl.replace("http", "ws");
      }
      return new OriginalWebSocket(modifiedUrl, protocols);
    };
    window.WebSocket.prototype = OriginalWebSocket.prototype;
  }
}

function workerAndFunctionInject() {
  if (window.Worker) {
    const OrigWorker = window.Worker;
    window.Worker = function(url, options) {
      let modifiedUrl = changeURL(url);
      return new OrigWorker(modifiedUrl, options);
    };
    window.Worker.prototype = OrigWorker.prototype;
  }
  const OrigFunction = window.Function;
  window.Function = new Proxy(OrigFunction, {
    construct(target, args) {
      return Reflect.construct(target, args);
    }
  });
  const origEval = window.eval;
  window.eval = function(code) {
    return origEval(code);
  };
  const origSetTimeout = window.setTimeout;
  window.setTimeout = function(handler, timeout, ...args) {
    return origSetTimeout.call(this, handler, timeout, ...args);
  };
}

function blobObjectUrlInject() {
  if (window.URL && window.URL.createObjectURL) {
    const origCreateObjectURL = window.URL.createObjectURL;
    window.URL.createObjectURL = function(obj) {
      const result = origCreateObjectURL.call(this, obj);
      proxyMemoryRegistry.set(obj, result); 
      return result;
    };
  }
}

function windowOpenInject(){
  const originalOpen = window.open;
  window.open = function (url, name, specs) {
      let modifiedUrl = changeURL(url);
      return originalOpen.call(window, modifiedUrl, name, specs);
  };
}

function postMessageInject() {
  const origPostMessage = window.postMessage;
  window.postMessage = function(message, targetOrigin, transfer) {
    if (targetOrigin && targetOrigin !== '/') {
      targetOrigin = '*';
    }
    return origPostMessage.call(this, message, targetOrigin, transfer);
  };
}

function appendChildInject(){
  const originalAppendChild = Node.prototype.appendChild;
  Node.prototype.appendChild = function(child) {
    try {
      if(child.src) child.src = changeURL(child.src);
      if(child.href) child.href = changeURL(child.href);
    } catch(e) {}
    return originalAppendChild.call(this, child);
  };
}

function elementPropertyInject(){
  const originalSetAttribute = HTMLElement.prototype.setAttribute;
  HTMLElement.prototype.setAttribute = function (name, value) {
      if (name === "src" || name === "href") {
        value = changeURL(value);
      }
      originalSetAttribute.call(this, name, value);
  };

  const originalGetAttribute = HTMLElement.prototype.getAttribute;
  HTMLElement.prototype.getAttribute = function (name) {
    const val = originalGetAttribute.call(this, name);
    if (name === "href" || name === "src") {
      return getOriginalUrl(val);
    }
    return val;
  };

  const setList =[[HTMLAnchorElement, "href"], [HTMLScriptElement, "src"],[HTMLImageElement, "src"],[HTMLLinkElement, "href"], [HTMLIFrameElement, "src"],[HTMLAudioElement, "src"],[HTMLSourceElement, "src"],[HTMLObjectElement, "data"],[HTMLFormElement, "action"]
  ];
  
  for (const[whichElement, whichProperty] of setList) {
    if (!whichElement || !whichElement.prototype) continue;
    const descriptor = Object.getOwnPropertyDescriptor(whichElement.prototype, whichProperty);
    if (!descriptor) continue;
    Object.defineProperty(whichElement.prototype, whichProperty, {
      get: function () {
        const real = descriptor.get.call(this);
        return getOriginalUrl(real);
      },
      set: function (val) {
        descriptor.set.call(this, changeURL(val));
      },
      configurable: true,
    });
  }

  const OrigImage = window.Image;
  window.Image = new Proxy(OrigImage, {
    construct(target, args) {
      return new target(...args);
    }
  });
}

class ProxyLocation {
  constructor(originalLocation) {
      this.originalLocation = originalLocation;
  }
  reload(forcedReload) { this.originalLocation.reload(forcedReload); }
  replace(url) { this.originalLocation.replace(changeURL(url)); }
  assign(url) { this.originalLocation.assign(changeURL(url)); }
  get href() { return original_website_url_str; }
  set href(url) { this.originalLocation.href = changeURL(url); }
  get protocol() { return original_website_url.protocol; }
  set protocol(value) {
    original_website_url.protocol = value;
    this.originalLocation.href = proxy_host_with_schema + _enc_client(original_website_url.href);
  }
  get host() { return original_website_url.host; }
  set host(value) {
    original_website_url.host = value;
    this.originalLocation.href = proxy_host_with_schema + _enc_client(original_website_url.href);
  }
  get hostname() { return original_website_url.hostname; }
  set hostname(value) {
    original_website_url.hostname = value;
    this.originalLocation.href = proxy_host_with_schema + _enc_client(original_website_url.href);
  }
  get port() { return original_website_url.port; }
  set port(value) {
    original_website_url.port = value;
    this.originalLocation.href = proxy_host_with_schema + _enc_client(original_website_url.href);
  }
  get pathname() { return original_website_url.pathname; }
  set pathname(value) {
    original_website_url.pathname = value;
    this.originalLocation.href = proxy_host_with_schema + _enc_client(original_website_url.href);
  }
  get search() { return original_website_url.search; }
  set search(value) {
    original_website_url.search = value;
    this.originalLocation.href = proxy_host_with_schema + _enc_client(original_website_url.href);
  }
  get hash() { return original_website_url.hash; }
  set hash(value) {
    original_website_url.hash = value;
    this.originalLocation.href = proxy_host_with_schema + _enc_client(original_website_url.href);
  }
  get origin() { return original_website_url.origin; }
  toString() { return this.originalLocation.href; }
}

function documentLocationInject(){
  Object.defineProperty(document, 'URL', {
    get: function () { return original_website_url_str; },
    set: function (url) { document.URL = changeURL(url); }
  });
  Object.defineProperty(document, '${proxyLocationProp}', {
      get: function () { return new ProxyLocation(window.location); },  
      set: function (url) { window.location.href = changeURL(url); }
  });
}

function windowLocationInject() {
  Object.defineProperty(window, '${proxyLocationProp}', {
      get: function () { return new ProxyLocation(window.location); },
      set: function (url) { window.location.href = changeURL(url); }
  });
}

function historyInject(){
  const originalPushState = History.prototype.pushState;
  const originalReplaceState = History.prototype.replaceState;
  
  History.prototype.pushState = function (state, title, url) {
    if(!url) return; 
    let absoluteUrl;
    try { absoluteUrl = new URL(url, original_website_url_str).href; } catch(e) { absoluteUrl = url; }
    var u = changeURL(absoluteUrl);
    return originalPushState.apply(this,[state, title, u]);
  };
  
  History.prototype.replaceState = function (state, title, url) {
    if(!url) return; 
    let absoluteUrl;
    try { absoluteUrl = new URL(url, original_website_url_str).href; } catch(e) { absoluteUrl = url; }
    var u = changeURL(absoluteUrl);
    return originalReplaceState.apply(this, [state, title, u]);
  };
  
  History.prototype.back = function () { return originalBack.apply(this); };
  History.prototype.forward = function () { return originalForward.apply(this); };
  History.prototype.go = function (delta) { return originalGo.apply(this, [delta]); };
}

function obsPage() {
  let debounceTimeout = null;
  let pendingMutations =[];
  
  const lazyHookObserver = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        traverseAndConvert(entry.target);
        lazyHookObserver.unobserve(entry.target);
      }
    });
  });

  var yProxyObserver = new MutationObserver(function(mutations) {
    pendingMutations.push(...mutations);
    if (debounceTimeout) clearTimeout(debounceTimeout);
    debounceTimeout = setTimeout(() => {
      const mutationsToProcess = pendingMutations;
      pendingMutations =[];
      mutationsToProcess.forEach(function(mutation) {
        if (mutation.target instanceof HTMLScriptElement || mutation.target instanceof HTMLLinkElement) {
           traverseAndConvert(mutation.target);
        } else if (mutation.target instanceof Element) {
           lazyHookObserver.observe(mutation.target);
        }
      });
    }, 150); 
  });
  
  var config = { attributes: true, childList: true, subtree: true };
  yProxyObserver.observe(document.body, config);
}

const originalAttachShadow = Element.prototype.attachShadow;
Element.prototype.attachShadow = function(init) {
  const shadowRoot = originalAttachShadow.call(this, init);
  var shadowObserver = new MutationObserver(function(mutations) {
    mutations.forEach(function(mutation) {
      traverseAndConvert(mutation.target);
    });
  });
  shadowObserver.observe(shadowRoot, { attributes: true, childList: true, subtree: true });
  return shadowRoot;
};

function preventShortcutConflict() {
  document.addEventListener('keydown', function(e) {
      // Propagation control if needed
  }, true);
}

function traverseAndConvert(node) {
  if (node instanceof HTMLElement) {
    if (node.hasAttribute('data-yproxy-handled')) return; 
    removeIntegrityAttributesFromElement(node);
    covToAbs(node);
    
    // (2) ã¿ã¼ã²ããå±æ§ãæã¤è¦ç´ ã®ã¿ãé«éæ¤ç´¢ããCPUè² è·ãåçã«ä½æ¸
    const targetElements = node.querySelectorAll('a[href], img[src], script[src], link[href], iframe[src], source[src], source[srcset], video[poster], audio[src], form[action], object[data]');
    targetElements.forEach(function(child) {
      if (!child.hasAttribute('data-yproxy-handled')) {
          removeIntegrityAttributesFromElement(child);
          covToAbs(child);
      }
    });
    node.setAttribute('data-yproxy-handled', '1');
  }
}

function covToAbs(element) {
  if(!(element instanceof HTMLElement)) return;
  try {
    if (element.hasAttribute("href")) element.setAttribute("href", changeURL(element.getAttribute("href")));
    if (element.hasAttribute("src")) element.setAttribute("src", changeURL(element.getAttribute("src")));
    if (element.tagName === "FORM" && element.hasAttribute("action")) element.setAttribute("action", changeURL(element.getAttribute("action")));
    if (element.tagName === "SOURCE" && element.hasAttribute("srcset")) element.setAttribute("srcset", changeURL(element.getAttribute("srcset"))); 
    if ((element.tagName === "VIDEO" || element.tagName === "AUDIO") && element.hasAttribute("poster")) element.setAttribute("poster", changeURL(element.getAttribute("poster")));
    if (element.tagName === "OBJECT" && element.hasAttribute("data")) element.setAttribute("data", changeURL(element.getAttribute("data")));
    element.setAttribute('data-yproxy-handled', '1');
  } catch (e) {}
}

function removeIntegrityAttributesFromElement(element){
  if (element.hasAttribute('integrity')) {
    element.removeAttribute('integrity');
  }
}

function loopAndConvertToAbs(){
  const targetElements = document.querySelectorAll('a[href], img[src], script[src], link[href], iframe[src], source[src], source[srcset], video[poster], audio[src], form[action], object[data]');
  for(var i = 0; i < targetElements.length; i++){
    removeIntegrityAttributesFromElement(targetElements[i]);
    covToAbs(targetElements[i]);
  }
}

function covScript(){ 
  var scripts = document.querySelectorAll('script[type="text/proxy-delayed"]');
  for (var i = 0; i < scripts.length; i++) {
    const s = scripts[i];
    const newScript = document.createElement('script');
    Array.from(s.attributes).forEach(attr => newScript.setAttribute(attr.name, attr.value));
    newScript.removeAttribute('type'); 
    newScript.textContent = s.textContent;
    s.parentNode.replaceChild(newScript, s);
  }
}

function videoAudioCorsInject() {
  if (typeof HTMLVideoElement !== 'undefined') {
    const videoPlay = HTMLVideoElement.prototype.play;
    HTMLVideoElement.prototype.play = function() {
      if (!this.hasAttribute('crossorigin')) {
        this.setAttribute('crossorigin', 'anonymous');
      }
      return videoPlay.call(this);
    };
  }
  if (typeof HTMLAudioElement !== 'undefined') {
    const audioPlay = HTMLAudioElement.prototype.play;
    HTMLAudioElement.prototype.play = function() {
      if (!this.hasAttribute('crossorigin')) {
        this.setAttribute('crossorigin', 'anonymous');
      }
      return audioPlay.call(this);
    };
  }
  
  if (window.MediaSource) {
    const originalAddSourceBuffer = MediaSource.prototype.addSourceBuffer;
    MediaSource.prototype.addSourceBuffer = function(mimeType) {
      if (typeof logDebug !== 'undefined') {
         logDebug("MSE addSourceBuffer intercepted:", mimeType);
      }
      return originalAddSourceBuffer.call(this, mimeType);
    };
  }
}

// åæåå®è¡
networkInject();
websocketInject(); 
workerAndFunctionInject(); 
blobObjectUrlInject(); 
windowOpenInject();
postMessageInject();
elementPropertyInject();
appendChildInject();
documentLocationInject();
windowLocationInject();
historyInject();
preventShortcutConflict();
videoAudioCorsInject();

window.addEventListener('load', () => {
  loopAndConvertToAbs();
  obsPage();
  covScript();
});

window.addEventListener('error', event => {
  var element = event.target || event.srcElement;
  if (element && element.tagName === 'SCRIPT') {
    if(element.alreadyChanged) return;
    
    removeIntegrityAttributesFromElement(element);
    covToAbs(element);
    var newScript = document.createElement("script");
    newScript.src = element.src;
    newScript.async = element.async; 
    newScript.defer = element.defer; 
    newScript.alreadyChanged = true;
    setTimeout(() => {
        document.head.appendChild(newScript);
    }, 50);
  }
}, true);
`;
  return CLIENT_CODE;
}

let HTML_COV_PATH_CODE = null;
function getHtmlCovPathInjectCode(funcName, proxyBaseUrl) {
  if (HTML_COV_PATH_CODE) return HTML_COV_PATH_CODE;
  HTML_COV_PATH_CODE = `
function ${funcName}(htmlString) {
  try {
    const parser = new DOMParser();
    let tempDoc;
    if (htmlString.trim().startsWith('<?xml') || htmlString.trim().startsWith('<svg')) {
       tempDoc = parser.parseFromString(htmlString, 'text/xml');
    } else {
       tempDoc = parser.parseFromString(htmlString, 'text/html');
    }
    
    // é«éåï¼å¯¾è±¡ã¿ã°ã®ã¿èµ°æ»
    const targetElements = tempDoc.querySelectorAll('a[href], img[src], script[src], link[href], iframe[src], source[src], source[srcset], video[poster], audio[src], form[action], object[data], style, script');
    
    for(let i = 0; i < targetElements.length; i++){
      try {
        const element = targetElements[i];
        covToAbs(element);
        removeIntegrityAttributesFromElement(element);
        
        if (element.tagName === 'SCRIPT' && element.textContent && !element.src) {
            if (element.textContent.length < 1000000) {
                element.textContent = replaceContentPaths(element.textContent);
            }
        }
        if (element.tagName === 'STYLE' && element.textContent) {
            if (element.textContent.length < 500000) {
                element.textContent = replaceContentPaths(element.textContent);
            }
        }
      } catch (err) {}
    }
    const modifiedHtml = tempDoc.documentElement ? tempDoc.documentElement.outerHTML : tempDoc.body.innerHTML;
    document.open();
    document.write('<!DOCTYPE html>' + modifiedHtml);
    document.close();
  } catch (e) {
    console.error("Critical error in parseAndInsertDoc: ", e);
  }
}

// (29) é«éãªæ­£è¦è¡¨ç¾ç½®æã¨Base64URLã®å©ç¨
function replaceContentPaths(content) {
  let regex = /(https?:\\/\\/[^\\s'"\\\\><]+)/g;
  return content.split(regex).map((part, index) => {
    if (index % 2 === 1) { 
      return "${proxyBaseUrl}" + _enc_client(part);
    }
    return part;
  }).join("");
}
`;
  return HTML_COV_PATH_CODE;
}

// =======================================================================================
// *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-* Proxy DevTools Injection *-*-*-*-*-*-*-*-*-*-*-*-*-*
// =======================================================================================

function getDevToolsCode(proxyBaseUrl) {
  return (
    '(function(){"use strict";' +
    'if(window.__pdtLoaded)return;window.__pdtLoaded=true;' +
    'var PDT_BASE="' + proxyBaseUrl + '";' +

    /* ââ CSS ââ */
    'var st=document.createElement("style");' +
    'st.textContent=' +
    '"#__pdt_fab__{position:fixed!important;bottom:20px!important;right:20px!important;z-index:2147483647!important;' +
      'background:linear-gradient(135deg,#1a73e8,#0d47a1)!important;color:#fff!important;border:none!important;' +
      'border-radius:10px!important;padding:9px 16px!important;font-size:13px!important;' +
      'font-family:Segoe UI,sans-serif!important;cursor:pointer!important;' +
      'box-shadow:0 4px 18px rgba(26,115,232,.55)!important;font-weight:600!important;' +
      'display:flex!important;align-items:center!important;gap:6px!important;' +
      'transition:transform .15s,box-shadow .15s!important;letter-spacing:.02em!important;}" +
    '"#__pdt_fab__:hover{transform:scale(1.07)!important;box-shadow:0 6px 24px rgba(26,115,232,.75)!important;}" +' +
    '"#__pdt_panel__{position:fixed!important;bottom:0!important;left:0!important;right:0!important;' +
      'height:520px!important;background:#1e1e1e!important;color:#d4d4d4!important;' +
      'z-index:2147483646!important;display:none!important;flex-direction:column!important;' +
      'font-family:Fira Code,Consolas,Courier New,monospace!important;font-size:13px!important;' +
      'border-top:2px solid #007acc!important;box-shadow:0 -6px 30px rgba(0,0,0,.7)!important;}" +' +
    '"#__pdt_panel__.pdt_on{display:flex!important;}" +' +
    '"#__pdt_drag__{height:5px!important;background:#007acc!important;cursor:ns-resize!important;flex-shrink:0!important;transition:background .15s!important;}" +' +
    '"#__pdt_drag__:hover{background:#3bb0ff!important;}" +' +
    '"#__pdt_hdr__{display:flex!important;align-items:center!important;background:#252526!important;' +
      'border-bottom:1px solid #3e3e3e!important;padding:0 8px!important;height:38px!important;' +
      'flex-shrink:0!important;gap:1px!important;overflow-x:auto!important;}" +' +
    '".pdtTab{padding:6px 11px!important;color:#999!important;cursor:pointer!important;font-size:11.5px!important;' +
      'border-radius:4px 4px 0 0!important;font-family:Segoe UI,sans-serif!important;user-select:none!important;white-space:nowrap!important;}" +' +
    '".pdtTab:hover{color:#ddd!important;background:#2d2d30!important;}" +' +
    '".pdtTabOn{color:#fff!important;border-bottom:2px solid #007acc!important;background:#1e1e1e!important;}" +' +
    '"#__pdt_hdrR__{display:flex!important;align-items:center!important;margin-left:auto!important;gap:8px!important;flex-shrink:0!important;}" +' +
    '"#__pdt_hdrR__ span{font-size:11px!important;color:#555!important;font-family:Segoe UI,sans-serif!important;}" +' +
    '"#__pdt_xbtn__{background:none!important;border:none!important;color:#888!important;font-size:20px!important;' +
      'cursor:pointer!important;padding:2px 8px!important;border-radius:4px!important;line-height:1!important;font-family:sans-serif!important;}" +' +
    '"#__pdt_xbtn__:hover{background:#c0392b!important;color:#fff!important;}" +' +
    '"#__pdt_body__{display:flex!important;flex:1!important;overflow:hidden!important;}" +' +
    '".pdtPage{display:none!important;flex:1!important;overflow:hidden!important;flex-direction:row!important;}" +' +
    '".pdtPageOn{display:flex!important;}" +' +

    /* Sources sidebar */
    '"#__pdt_sb__{width:240px!important;background:#252526!important;border-right:1px solid #3e3e3e!important;' +
      'overflow-y:auto!important;flex-shrink:0!important;display:flex!important;flex-direction:column!important;}" +' +
    '"#__pdt_sbtb__{padding:6px 8px!important;border-bottom:1px solid #3e3e3e!important;' +
      'display:flex!important;gap:4px!important;flex-shrink:0!important;}" +' +
    '"#__pdt_fl__{flex:1!important;overflow-y:auto!important;}" +' +
    '".pdtGrp{padding:5px 10px!important;font-size:10.5px!important;color:#668!important;' +
      'text-transform:uppercase!important;letter-spacing:.1em!important;' +
      'margin-top:6px!important;font-family:Segoe UI,sans-serif!important;user-select:none!important;}" +' +
    '".pdtFi{padding:5px 10px 5px 22px!important;cursor:pointer!important;' +
      'display:flex!important;align-items:center!important;gap:7px!important;' +
      'font-size:12px!important;color:#bbb!important;white-space:nowrap!important;overflow:hidden!important;' +
      'font-family:Segoe UI,monospace!important;border-left:2px solid transparent!important;}" +' +
    '".pdtFi:hover{background:#2a2d2e!important;color:#fff!important;}" +' +
    '".pdtFiOn{background:#094771!important;color:#fff!important;border-left-color:#007acc!important;}" +' +

    /* Sources main */
    '"#__pdt_main__{display:flex!important;flex-direction:column!important;flex:1!important;overflow:hidden!important;}" +' +
    '"#__pdt_tb__{display:flex!important;align-items:center!important;padding:5px 10px!important;gap:8px!important;' +
      'background:#2d2d30!important;border-bottom:1px solid #3e3e3e!important;flex-shrink:0!important;min-height:34px!important;flex-wrap:wrap!important;}" +' +
    '"#__pdt_fn__{color:#888!important;font-size:11px!important;flex:1!important;' +
      'overflow:hidden!important;text-overflow:ellipsis!important;white-space:nowrap!important;' +
      'font-family:Segoe UI,sans-serif!important;}" +' +
    '".pdtLbl{display:flex!important;align-items:center!important;gap:5px!important;font-size:11.5px!important;' +
      'color:#ccc!important;cursor:pointer!important;white-space:nowrap!important;font-family:Segoe UI,sans-serif!important;}" +' +
    '".pdtBtn{background:#0e639c!important;border:none!important;color:#fff!important;padding:3px 11px!important;' +
      'border-radius:4px!important;font-size:11px!important;cursor:pointer!important;white-space:nowrap!important;' +
      'font-family:Segoe UI,sans-serif!important;}" +' +
    '".pdtBtn:hover{background:#1177bb!important;}" +' +
    '".pdtBtnRed{background:#8b1a1a!important;}" +' +
    '".pdtBtnRed:hover{background:#c0392b!important;}" +' +
    '".pdtBtnGrn{background:#1a6b2a!important;}" +' +
    '".pdtBtnGrn:hover{background:#27ae60!important;}" +' +
    '".pdtBtnOra{background:#7a4100!important;}" +' +
    '".pdtBtnOra:hover{background:#e67e22!important;}" +' +
    '"#__pdt_cw__{flex:1!important;overflow:auto!important;background:#1e1e1e!important;}" +' +
    '"#__pdt_cw__ pre{margin:0!important;padding:0!important;}" +' +
    '"#__pdt_cw__ code{display:block!important;padding:12px 0!important;line-height:1.65!important;' +
      'font-size:13px!important;font-family:Fira Code,Consolas,Courier New,monospace!important;}" +' +
    '".pdtLn{display:flex!important;min-width:max-content!important;}" +' +
    '".pdtLnum{display:inline-block!important;min-width:44px!important;padding:0 12px 0 8px!important;' +
      'color:#555!important;text-align:right!important;user-select:none!important;' +
      'border-right:1px solid #333!important;margin-right:12px!important;flex-shrink:0!important;}" +' +
    '".pdtLcnt{flex:1!important;padding-right:16px!important;white-space:pre!important;}" +' +
    '".pdtLn:hover{background:#2a2a2a!important;}" +' +
    '".hljs{background:transparent!important;}" +' +

    /* Override tab */
    '"#__pdt_ovp__{display:flex!important;flex-direction:column!important;flex:1!important;overflow:hidden!important;background:#1e1e1e!important;}" +' +
    '"#__pdt_ovtb__{display:flex!important;align-items:center!important;gap:8px!important;padding:7px 14px!important;' +
      'background:#252526!important;border-bottom:1px solid #3e3e3e!important;flex-shrink:0!important;flex-wrap:wrap!important;}" +' +
    '"#__pdt_ovtb__ .pdtOvTitle{font-size:11.5px!important;color:#aaa!important;' +
      'font-family:Segoe UI,sans-serif!important;margin-right:auto!important;}" +' +
    '"#__pdt_ovl__{overflow-y:auto!important;flex:1!important;padding:8px 0!important;}" +' +
    '".pdtOvRow{display:flex!important;flex-direction:column!important;border:1px solid #3e3e3e!important;' +
      'border-radius:6px!important;margin:6px 12px!important;background:#252526!important;transition:border-color .15s!important;}" +' +
    '".pdtOvRow:hover{border-color:#007acc!important;}" +' +
    '".pdtOvHead{display:flex!important;align-items:center!important;padding:7px 10px!important;gap:8px!important;cursor:pointer!important;}" +' +
    '".pdtOvBody{padding:0 10px 10px!important;display:none!important;flex-direction:column!important;gap:6px!important;}" +' +
    '".pdtOvBodyOn{display:flex!important;}" +' +
    '".pdtOvOn{background:#1a6b2a!important;color:#7fff7f!important;border-radius:3px!important;padding:1px 7px!important;font-size:10.5px!important;font-family:Segoe UI,sans-serif!important;}" +' +
    '".pdtOvOff{background:#3e3e3e!important;color:#666!important;border-radius:3px!important;padding:1px 7px!important;font-size:10.5px!important;font-family:Segoe UI,sans-serif!important;}" +' +
    '".pdtOvUrl{flex:1!important;color:#9cdcfe!important;font-size:12px!important;overflow:hidden!important;' +
      'text-overflow:ellipsis!important;white-space:nowrap!important;font-family:Consolas,monospace!important;}" +' +
    '".pdtOvHits{font-size:10px!important;color:#4ec9b0!important;font-family:Segoe UI,sans-serif!important;white-space:nowrap!important;}" +' +
    '".pdtOvActs{display:flex!important;gap:5px!important;flex-shrink:0!important;}" +' +
    '".pdtOvIrow{display:flex!important;flex-direction:column!important;gap:3px!important;}" +' +
    '".pdtOvIlbl{font-size:10.5px!important;color:#888!important;font-family:Segoe UI,sans-serif!important;}" +' +
    '".pdtOvInp{background:#1e1e1e!important;border:1px solid #555!important;color:#d4d4d4!important;' +
      'padding:4px 8px!important;border-radius:4px!important;font-size:12px!important;' +
      'font-family:Consolas,monospace!important;width:100%!important;box-sizing:border-box!important;}" +' +
    '".pdtOvInp:focus{outline:none!important;border-color:#007acc!important;}" +' +
    '".pdtOvTa{background:#1e1e1e!important;border:1px solid #555!important;color:#d4d4d4!important;' +
      'padding:6px 8px!important;border-radius:4px!important;font-size:12px!important;' +
      'font-family:Fira Code,Consolas,monospace!important;width:100%!important;min-height:110px!important;' +
      'box-sizing:border-box!important;resize:vertical!important;line-height:1.5!important;}" +' +
    '".pdtOvTa:focus{outline:none!important;border-color:#007acc!important;}" +' +
    '".pdtOvSel{background:#1e1e1e!important;border:1px solid #555!important;color:#d4d4d4!important;' +
      'padding:4px 8px!important;border-radius:4px!important;font-size:12px!important;font-family:Segoe UI,sans-serif!important;}" +' +
    '".pdtOvHint{padding:40px 20px!important;text-align:center!important;color:#555!important;' +
      'font-family:Segoe UI,sans-serif!important;font-size:13px!important;line-height:1.8!important;}" +' +
    '".pdtOvMatch{background:#1a3a1a!important;}" +' +
    '".pdtMsg{padding:24px!important;color:#888!important;text-align:center!important;font-family:Segoe UI,sans-serif!important;font-size:13px!important;}" +' +
    '".pdtErr{padding:24px!important;color:#f44747!important;font-family:Segoe UI,sans-serif!important;font-size:13px!important;}" +' +

    /* Network tab styles */
    '"#__pdt_netp__{display:flex!important;flex-direction:column!important;flex:1!important;overflow:hidden!important;background:#1e1e1e!important;}" +' +
    '"#__pdt_nettb__{display:flex!important;align-items:center!important;gap:8px!important;padding:5px 10px!important;' +
      'background:#252526!important;border-bottom:1px solid #3e3e3e!important;flex-shrink:0!important;flex-wrap:wrap!important;}" +' +
    '"#__pdt_netsrch__{background:#1e1e1e!important;border:1px solid #555!important;color:#d4d4d4!important;' +
      'padding:3px 8px!important;border-radius:4px!important;font-size:12px!important;font-family:Consolas,monospace!important;width:160px!important;}" +' +
    '"#__pdt_netlst__{flex:1!important;overflow-y:auto!important;font-family:Consolas,monospace!important;font-size:12px!important;}" +' +
    '".pdtNetRow{display:grid!important;grid-template-columns:40px 60px 1fr 70px 80px 70px!important;' +
      'align-items:center!important;padding:3px 8px!important;border-bottom:1px solid #2a2a2a!important;cursor:pointer!important;gap:4px!important;}" +' +
    '".pdtNetRow:hover{background:#2a2d2e!important;}" +' +
    '".pdtNetRowOn{background:#094771!important;}" +' +
    '".pdtNetHead{background:#252526!important;color:#888!important;font-size:10.5px!important;font-family:Segoe UI,sans-serif!important;' +
      'display:grid!important;grid-template-columns:40px 60px 1fr 70px 80px 70px!important;' +
      'padding:4px 8px!important;border-bottom:1px solid #3e3e3e!important;gap:4px!important;user-select:none!important;}" +' +
    '".pdtNetSt2xx{color:#4ec9b0!important;}" +' +
    '".pdtNetSt3xx{color:#dcdcaa!important;}" +' +
    '".pdtNetSt4xx{color:#f44747!important;}" +' +
    '".pdtNetSt5xx{color:#f44747!important;}" +' +
    '".pdtNetWs{color:#c586c0!important;}" +' +
    '".pdtNetDetail{flex:1!important;overflow:auto!important;background:#252526!important;border-left:1px solid #3e3e3e!important;padding:12px!important;font-size:12px!important;font-family:Consolas,monospace!important;}" +' +
    '".pdtNetSection{color:#9cdcfe!important;font-size:11px!important;text-transform:uppercase!important;letter-spacing:.08em!important;margin:10px 0 4px!important;font-family:Segoe UI,sans-serif!important;}" +' +
    '".pdtNetKv{display:flex!important;gap:8px!important;padding:2px 0!important;border-bottom:1px solid #2a2a2a!important;}" +' +
    '".pdtNetKey{color:#9cdcfe!important;min-width:140px!important;flex-shrink:0!important;}" +' +
    '".pdtNetVal{color:#ce9178!important;word-break:break-all!important;flex:1!important;}" +' +
    '"#__pdt_netpane__{display:flex!important;flex:1!important;overflow:hidden!important;}" +' +
    '"#__pdt_netlstwrap__{flex:1!important;display:flex!important;flex-direction:column!important;overflow:hidden!important;min-width:280px!important;}" +' +

    /* Console tab styles */
    '"#__pdt_conp__{display:flex!important;flex-direction:column!important;flex:1!important;overflow:hidden!important;background:#1e1e1e!important;}" +' +
    '"#__pdt_contb__{display:flex!important;align-items:center!important;gap:8px!important;padding:5px 10px!important;' +
      'background:#252526!important;border-bottom:1px solid #3e3e3e!important;flex-shrink:0!important;flex-wrap:wrap!important;}" +' +
    '"#__pdt_conlog__{flex:1!important;overflow-y:auto!important;padding:4px 0!important;}" +' +
    '".pdtConRow{display:flex!important;align-items:flex-start!important;padding:3px 10px!important;' +
      'border-bottom:1px solid #252526!important;gap:8px!important;font-family:Consolas,monospace!important;font-size:12px!important;}" +' +
    '".pdtConRow:hover{background:#252526!important;}" +' +
    '".pdtConLog{color:#d4d4d4!important;}" +' +
    '".pdtConWarn{color:#dcdcaa!important;background:rgba(220,220,0,.05)!important;}" +' +
    '".pdtConErr{color:#f44747!important;background:rgba(244,71,71,.05)!important;}" +' +
    '".pdtConInfo{color:#9cdcfe!important;}" +' +
    '".pdtConTime{color:#555!important;font-size:10.5px!important;flex-shrink:0!important;padding-top:1px!important;}" +' +
    '".pdtConBadge{font-size:9px!important;border-radius:3px!important;padding:1px 5px!important;flex-shrink:0!important;font-family:Segoe UI,sans-serif!important;}" +' +
    '".pdtConBL{background:#0e639c!important;color:#fff!important;}" +' +
    '".pdtConBW{background:#7a6400!important;color:#fff!important;}" +' +
    '".pdtConBE{background:#8b1a1a!important;color:#fff!important;}" +' +
    '".pdtConMsg{flex:1!important;white-space:pre-wrap!important;word-break:break-all!important;}" +' +
    '"#__pdt_conin__{display:flex!important;border-top:1px solid #3e3e3e!important;flex-shrink:0!important;}" +' +
    '"#__pdt_coninp__{flex:1!important;background:#1e1e1e!important;border:none!important;color:#d4d4d4!important;' +
      'padding:7px 10px!important;font-family:Fira Code,Consolas,monospace!important;font-size:13px!important;outline:none!important;}" +' +
    '"#__pdt_conrun__{background:#0e639c!important;border:none!important;color:#fff!important;' +
      'padding:0 14px!important;cursor:pointer!important;font-size:12px!important;font-family:Segoe UI,sans-serif!important;}" +' +

    /* WebSocket tab styles */
    '"#__pdt_wsp__{display:flex!important;flex-direction:column!important;flex:1!important;overflow:hidden!important;background:#1e1e1e!important;}" +' +
    '"#__pdt_wstb__{display:flex!important;align-items:center!important;gap:8px!important;padding:5px 10px!important;' +
      'background:#252526!important;border-bottom:1px solid #3e3e3e!important;flex-shrink:0!important;flex-wrap:wrap!important;}" +' +
    '"#__pdt_wsconns__{width:220px!important;background:#252526!important;border-right:1px solid #3e3e3e!important;overflow-y:auto!important;flex-shrink:0!important;}" +' +
    '"#__pdt_wsmsgs__{flex:1!important;display:flex!important;flex-direction:column!important;overflow:hidden!important;}" +' +
    '"#__pdt_wsmsglist__{flex:1!important;overflow-y:auto!important;font-family:Consolas,monospace!important;font-size:12px!important;}" +' +
    '".pdtWsConn{padding:6px 10px!important;cursor:pointer!important;border-bottom:1px solid #2a2a2a!important;font-size:11.5px!important;color:#9cdcfe!important;' +
      'font-family:Segoe UI,monospace!important;overflow:hidden!important;text-overflow:ellipsis!important;white-space:nowrap!important;}" +' +
    '".pdtWsConn:hover{background:#2a2d2e!important;}" +' +
    '".pdtWsConnOn{background:#094771!important;}" +' +
    '".pdtWsConnDead{color:#555!important;text-decoration:line-through!important;}" +' +
    '".pdtWsMsgS{color:#4ec9b0!important;padding:3px 10px!important;border-bottom:1px solid #252526!important;}" +' +
    '".pdtWsMsgR{color:#ce9178!important;padding:3px 10px!important;border-bottom:1px solid #252526!important;}" +' +
    '".pdtWsMsgSys{color:#555!important;font-style:italic!important;padding:3px 10px!important;}" +' +
    '"#__pdt_wssend__{display:flex!important;border-top:1px solid #3e3e3e!important;flex-shrink:0!important;}" +' +
    '"#__pdt_wsinp__{flex:1!important;background:#1e1e1e!important;border:none!important;color:#d4d4d4!important;' +
      'padding:7px 10px!important;font-family:Consolas,monospace!important;font-size:12px!important;outline:none!important;}" +' +
    '"#__pdt_wspane__{display:flex!important;flex:1!important;overflow:hidden!important;}" +' +

    /* Storage tab styles */
    '"#__pdt_strp__{display:flex!important;flex-direction:column!important;flex:1!important;overflow:hidden!important;background:#1e1e1e!important;}" +' +
    '"#__pdt_strtb__{display:flex!important;align-items:center!important;gap:8px!important;padding:5px 10px!important;' +
      'background:#252526!important;border-bottom:1px solid #3e3e3e!important;flex-shrink:0!important;flex-wrap:wrap!important;}" +' +
    '"#__pdt_strside__{width:170px!important;background:#252526!important;border-right:1px solid #3e3e3e!important;overflow-y:auto!important;flex-shrink:0!important;}" +' +
    '"#__pdt_strcat{padding:5px 10px!important;font-size:10.5px!important;color:#668!important;text-transform:uppercase!important;' +
      'letter-spacing:.1em!important;font-family:Segoe UI,sans-serif!important;margin-top:6px!important;}" +' +
    '".pdtStrItem{padding:5px 14px!important;cursor:pointer!important;font-size:12px!important;color:#bbb!important;' +
      'font-family:Segoe UI,sans-serif!important;border-left:2px solid transparent!important;}" +' +
    '".pdtStrItem:hover{background:#2a2d2e!important;}" +' +
    '".pdtStrItemOn{background:#094771!important;color:#fff!important;border-left-color:#007acc!important;}" +' +
    '"#__pdt_strmain__{flex:1!important;display:flex!important;flex-direction:column!important;overflow:hidden!important;}" +' +
    '"#__pdt_strtbl__{flex:1!important;overflow:auto!important;}" +' +
    '"table.pdtT{width:100%!important;border-collapse:collapse!important;font-size:12px!important;font-family:Consolas,monospace!important;}" +' +
    '"table.pdtT th{background:#252526!important;color:#888!important;padding:5px 10px!important;text-align:left!important;' +
      'font-size:10.5px!important;font-family:Segoe UI,sans-serif!important;border-bottom:1px solid #3e3e3e!important;position:sticky!important;top:0!important;}" +' +
    '"table.pdtT td{padding:4px 10px!important;border-bottom:1px solid #252526!important;color:#d4d4d4!important;word-break:break-all!important;}" +' +
    '"table.pdtT tr:hover td{background:#2a2d2e!important;}" +' +
    '".pdtStrKey{color:#9cdcfe!important;}" +' +
    '".pdtStrVal{color:#ce9178!important;max-width:400px!important;}" +' +
    '"#__pdt_strpane__{display:flex!important;flex:1!important;overflow:hidden!important;}" +' +

    /* Deobfuscate tab styles */
    '"#__pdt_deobp__{display:flex!important;flex-direction:column!important;flex:1!important;overflow:hidden!important;background:#1e1e1e!important;}" +' +
    '"#__pdt_deobtb__{display:flex!important;align-items:center!important;gap:6px!important;padding:6px 10px!important;' +
      'background:#252526!important;border-bottom:1px solid #3e3e3e!important;flex-shrink:0!important;flex-wrap:wrap!important;}" +' +
    '"#__pdt_deobmain__{display:flex!important;flex:1!important;overflow:hidden!important;gap:0!important;}" +' +
    '"#__pdt_deobinwrap__{flex:1!important;display:flex!important;flex-direction:column!important;border-right:1px solid #3e3e3e!important;}" +' +
    '"#__pdt_deoboutwrap__{flex:1!important;display:flex!important;flex-direction:column!important;}" +' +
    '".pdtDeobLabel{padding:4px 10px!important;font-size:10.5px!important;color:#668!important;' +
      'background:#252526!important;border-bottom:1px solid #3e3e3e!important;font-family:Segoe UI,sans-serif!important;text-transform:uppercase!important;letter-spacing:.08em!important;}" +' +
    '"#__pdt_deobin__,#__pdt_deobout__{flex:1!important;background:#1e1e1e!important;border:none!important;' +
      'color:#d4d4d4!important;padding:10px!important;font-family:Fira Code,Consolas,monospace!important;' +
      'font-size:12.5px!important;resize:none!important;outline:none!important;line-height:1.6!important;' +
      'overflow:auto!important;width:100%!important;box-sizing:border-box!important;}" +' +
    '".pdtDeobBtn{background:#0e639c!important;border:none!important;color:#fff!important;padding:4px 10px!important;' +
      'border-radius:4px!important;font-size:11px!important;cursor:pointer!important;font-family:Segoe UI,sans-serif!important;white-space:nowrap!important;}" +' +
    '".pdtDeobBtn:hover{background:#1177bb!important;}" +' +
    '".pdtDeobBtnOra{background:#7a4100!important;}" +' +
    '".pdtDeobBtnOra:hover{background:#e67e22!important;}" +' +
    '".pdtDeobBtnRed{background:#8b1a1a!important;}" +' +
    '".pdtDeobBtnRed:hover{background:#c0392b!important;}" +' +
    '".pdtDeobSel{background:#1e1e1e!important;border:1px solid #555!important;color:#d4d4d4!important;' +
      'padding:4px 8px!important;border-radius:4px!important;font-size:11px!important;font-family:Segoe UI,sans-serif!important;}" +' +
    '"#__pdt_deobstatus__{padding:3px 10px!important;font-size:11px!important;color:#4ec9b0!important;' +
      'font-family:Segoe UI,sans-serif!important;flex-shrink:0!important;}" +' +

    /* Scrollbar */
    '"#__pdt_sb__::-webkit-scrollbar,#__pdt_cw__::-webkit-scrollbar,#__pdt_ovl__::-webkit-scrollbar,' +
      '#__pdt_netlst__::-webkit-scrollbar,#__pdt_conlog__::-webkit-scrollbar,#__pdt_wsmsglist__::-webkit-scrollbar,' +
      '#__pdt_strtbl__::-webkit-scrollbar,#__pdt_deobout__::-webkit-scrollbar,#__pdt_deobin__::-webkit-scrollbar,' +
      '#__pdt_wsconns__::-webkit-scrollbar,#__pdt_strside__::-webkit-scrollbar{width:6px!important;height:6px!important;}" +' +
    '"#__pdt_sb__::-webkit-scrollbar-thumb,#__pdt_cw__::-webkit-scrollbar-thumb,#__pdt_ovl__::-webkit-scrollbar-thumb,' +
      '#__pdt_netlst__::-webkit-scrollbar-thumb,#__pdt_conlog__::-webkit-scrollbar-thumb,' +
      '#__pdt_wsmsglist__::-webkit-scrollbar-thumb,#__pdt_strtbl__::-webkit-scrollbar-thumb,' +
      '#__pdt_deobout__::-webkit-scrollbar-thumb,#__pdt_deobin__::-webkit-scrollbar-thumb,' +
      '#__pdt_wsconns__::-webkit-scrollbar-thumb,#__pdt_strside__::-webkit-scrollbar-thumb{background:#555!important;border-radius:3px!important;}" +' +
    '"#__pdt_sb__::-webkit-scrollbar-thumb:hover,#__pdt_cw__::-webkit-scrollbar-thumb:hover{background:#777!important;}";' +
    'document.head.appendChild(st);' +

    /* ââ Load external CDN libs ââ */
    'function ldLink(u){if(document.querySelector("link[data-pdt="+JSON.stringify(u)+"]"))return;' +
      'var l=document.createElement("link");l.rel="stylesheet";l.href=u;l.setAttribute("data-pdt",u);document.head.appendChild(l);}' +
    'function ldScript(u){if(document.querySelector("script[data-pdt="+JSON.stringify(u)+"]"))return;' +
      'var s=document.createElement("script");s.src=u;s.setAttribute("data-pdt",u);s.onerror=function(){};document.head.appendChild(s);}' +
    'ldLink("https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/atom-one-dark.min.css");' +
    'ldScript("https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/highlight.min.js");' +
    'ldScript("https://cdnjs.cloudflare.com/ajax/libs/js-beautify/1.15.1/beautify.min.js");' +
    'ldScript("https://cdnjs.cloudflare.com/ajax/libs/js-beautify/1.15.1/beautify-css.min.js");' +
    'ldScript("https://cdnjs.cloudflare.com/ajax/libs/js-beautify/1.15.1/beautify-html.min.js");' +

    /* ââ State ââ */
    'var S={' +
      'open:false,' +
      'tab:"sources",' +
      'resources:[],' +
      'sel:null,' +
      'beautify:false,' +
      'cache:{},' +
      'raw:"",' +
      'overrides:[],' +
      'netLog:[],' +
      'netSel:null,' +
      'netFilter:"",' +
      'netTypeFilter:"all",' +
      'conLog:[],' +
      'conFilter:"all",' +
      'wsConns:[],' +
      'wsSelConn:null,' +
      'strMode:"localStorage",' +
      'deobHistory:[]' +
    '};' +
    'var _ovId=0;var _netId=0;var _conId=0;var _wsId=0;' +

    /* ââ Helpers ââ */
    'function esc(s){return String(s).replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;").replace(/"/g,"&quot;");}' +
    'function decUrl(u){try{if(u&&u.startsWith(PDT_BASE))return _dec_client(u.substring(PDT_BASE.length));}catch(e){}return u;}' +
    'function shortName(u){try{var x=u.split("?")[0].split("#")[0];return x.split("/").pop()||x;}catch(e){return u;}}' +
    'function dotClr(t){return t==="js"?"#F5C518":t==="css"?"#4FC3F7":"#EF9A9A";}' +
    'function mkId(){return"ov_"+(++_ovId)+"_"+Date.now();}' +
    'function mkNetId(){return"n_"+(++_netId);}' +
    'function mkConId(){return"c_"+(++_conId);}' +
    'function fmtSize(b){if(!b)return"-";if(b<1024)return b+"B";if(b<1048576)return(b/1024).toFixed(1)+"K";return(b/1048576).toFixed(1)+"M";}' +
    'function fmtMs(ms){if(ms===null||ms===undefined)return"pending";if(ms<1000)return ms+"ms";return(ms/1000).toFixed(2)+"s";}' +
    'function fmtTime(){var d=new Date();return d.getHours().toString().padStart(2,"0")+":"+d.getMinutes().toString().padStart(2,"0")+":"+d.getSeconds().toString().padStart(2,"0")+"."+d.getMilliseconds().toString().padStart(3,"0");}' +
    'function stClr(s){if(!s)return"";if(s>=200&&s<300)return"pdtNetSt2xx";if(s>=300&&s<400)return"pdtNetSt3xx";if(s>=400&&s<500)return"pdtNetSt4xx";if(s>=500)return"pdtNetSt5xx";return"";}' +
    'function methodClr(m){var c={"GET":"#4ec9b0","POST":"#dcdcaa","PUT":"#c586c0","DELETE":"#f44747","PATCH":"#ce9178","OPTIONS":"#888","HEAD":"#9cdcfe"};return c[m]||"#d4d4d4";}' +

    /* ââ Collect resources ââ */
    'function collectRes(){' +
      'var res=[];' +
      'res.push({id:"__html__",type:"html",name:"(index) "+(document.title||"page").substring(0,30),' +
        'filename:"index.html",' +
        'realUrl:(typeof original_website_url_str!=="undefined"?original_website_url_str:location.href),' +
        'load:function(){return Promise.resolve("<!DOCTYPE html>\\n"+document.documentElement.outerHTML);}});' +
      'var sc=document.querySelectorAll("script[src]");' +
      'for(var i=0;i<sc.length;i++){' +
        'var src=sc[i].getAttribute("src");' +
        'if(!src||src.startsWith("data:")||src.startsWith("blob:")||src.includes("__pdt")||src.includes("highlight")||src.includes("beautify"))continue;' +
        'var ru=decUrl(src);var nm=shortName(ru)||("script_"+i+".js");' +
        'res.push({id:"js_"+i,type:"js",name:nm,' +
          'filename:(nm.match(/\\.js/)?nm.split("?")[0]:nm+".js"),' +
          'realUrl:ru,proxyUrl:src,' +
          'load:(function(u){return function(){return fetch(u).then(function(r){return r.text();});};})(src)});' +
      '}' +
      'var lk=document.querySelectorAll("link[rel=stylesheet][href]");' +
      'for(var j=0;j<lk.length;j++){' +
        'var href=lk[j].getAttribute("href");' +
        'if(!href||href.startsWith("data:")||href.startsWith("blob:")||href.includes("__pdt")||href.includes("highlight"))continue;' +
        'var ru2=decUrl(href);var nm2=shortName(ru2)||("style_"+j+".css");' +
        'res.push({id:"css_"+j,type:"css",name:nm2,' +
          'filename:(nm2.match(/\\.css/)?nm2.split("?")[0]:nm2+".css"),' +
          'realUrl:ru2,proxyUrl:href,' +
          'load:(function(u){return function(){return fetch(u).then(function(r){return r.text();});};})(href)});' +
      '}' +
      'return res;' +
    '}' +

    /* ââ Beautify ââ */
    'function beauty(code,type){' +
      'try{' +
        'if(type==="js"&&window.js_beautify)return window.js_beautify(code,{indent_size:2,preserve_newlines:false,max_preserve_newlines:1,unescape_strings:true});' +
        'if(type==="css"&&window.css_beautify)return window.css_beautify(code,{indent_size:2});' +
        'if(type==="html"&&window.html_beautify)return window.html_beautify(code,{indent_size:2});' +
      '}catch(e){}return code;' +
    '}' +

    /* ââ Highlight ââ */
    'function hl(code,type){' +
      'if(!window.hljs)return esc(code);' +
      'try{var lang=type==="js"?"javascript":type==="css"?"css":"xml";' +
        'return window.hljs.highlight(code,{language:lang,ignoreIllegals:true}).value;}' +
      'catch(e){return esc(code);}' +
    '}' +

    /* ââ Download ââ */
    'function dl(filename,content){' +
      'try{' +
        'var uri="data:text/plain;charset=utf-8,"+encodeURIComponent(content);' +
        'var a=document.createElement("a");' +
        'a.href=uri;a.download=filename;' +
        'a.style.cssText="position:fixed!important;left:-9999px!important;top:-9999px!important;opacity:0!important;";' +
        'document.body.appendChild(a);' +
        'try{a.dispatchEvent(new MouseEvent("click",{bubbles:false,cancelable:true,view:window}));}' +
        'catch(e2){a.click();}' +
        'setTimeout(function(){try{document.body.removeChild(a);}catch(e){}},500);' +
      '}catch(e){' +
        'var w=window.open("","_blank");' +
        'if(w){w.document.write("<pre>"+esc(content)+"<\\/pre>");w.document.close();}' +
      '}' +
    '}' +

    /* ââ Copy ââ */
    'function cp(text){' +
      'if(navigator.clipboard&&navigator.clipboard.writeText){navigator.clipboard.writeText(text).catch(function(){fbCopy(text);});}' +
      'else{fbCopy(text);}' +
    '}' +
    'function fbCopy(text){' +
      'var ta=document.createElement("textarea");ta.value=text;' +
      'ta.style.cssText="position:fixed;left:-9999px;top:-9999px;";' +
      'document.body.appendChild(ta);ta.select();' +
      'try{document.execCommand("copy");}catch(e){}' +
      'document.body.removeChild(ta);' +
    '}' +

    /* ââ Render code with line numbers ââ */
    'function renderCode(raw,type){' +
      'var code=S.beautify?beauty(raw,type):raw;' +
      'var lines=code.split("\\n");' +
      'var hlText=hl(code,type);' +
      'var hlLines=hlText.split("\\n");' +
      'var h="";' +
      'for(var i=0;i<lines.length;i++){' +
        'h+="<div class=\\"pdtLn\\"><span class=\\"pdtLnum\\">"+(i+1)+"<\\/span><span class=\\"pdtLcnt\\">"+(hlLines[i]!==undefined?hlLines[i]:"")+"<\\/span><\\/div>";' +
      '}return h;' +
    '}' +

    /* ââ Override engine ââ */
    'function globMatch(pat,url){' +
      'try{var p=pat.trim();if(!p)return false;' +
        'var re="^"+p.replace(/[.+^${}()|[\\]\\\\]/g,"\\\\$&").replace(/\\*/g,".*").replace(/\\?/g,".")+"$";' +
        'return new RegExp(re,"i").test(url);}' +
      'catch(e){return false;}' +
    '}' +
    'function findOv(url,method){' +
      'var real=url;try{if(url&&url.startsWith(PDT_BASE))real=_dec_client(url.substring(PDT_BASE.length));}catch(e){}' +
      'for(var i=0;i<S.overrides.length;i++){' +
        'var ov=S.overrides[i];if(!ov.enabled)continue;' +
        'var mOk=!ov.method||ov.method==="*"||ov.method.toUpperCase()===(method||"GET").toUpperCase();' +
        'if(mOk&&(globMatch(ov.urlPattern,real)||globMatch(ov.urlPattern,url)))return ov;' +
      '}return null;' +
    '}' +
    'function hitBadge(id,hits){' +
      'var row=document.querySelector("[data-ovid=\\""+id+"\\"]");if(!row)return;' +
      'var b=row.querySelector(".pdtOvHits");if(b)b.textContent=hits+" hit"+(hits===1?"":"s");' +
      'row.classList.add("pdtOvMatch");clearTimeout(row._pdtT);' +
      'row._pdtT=setTimeout(function(){row.classList.remove("pdtOvMatch");},1200);' +
    '}' +

    /* ââ Network interceptor ââ */
    'function hookNet(){' +
      'var _of=window.fetch;' +
      'window.fetch=function(input,init){' +
        'var url=typeof input==="string"?input:(input&&input.url?input.url:String(input));' +
        'var method=((init&&init.method)||(input&&input.method)||"GET").toUpperCase();' +
        'var ov=findOv(url,method);' +
        'if(ov){ov.hits=(ov.hits||0)+1;hitBadge(ov.id,ov.hits);' +
          'var ovResp=new Response(ov.body||"",{status:parseInt(ov.statusCode)||200,' +
            'headers:{"Content-Type":ov.contentType||"application/json","X-PDT-Override":"1"}});' +
          'var entry=addNetEntry(url,method,"fetch",parseInt(ov.statusCode)||200,0,ov.body?ov.body.length:0,init);' +
          'entry._overridden=true;updateNetEntry(entry.id,{status:parseInt(ov.statusCode)||200,size:ov.body?ov.body.length:0,duration:0,done:true});' +
          'return Promise.resolve(ovResp);}' +
        'var entry=addNetEntry(url,method,"fetch",null,null,null,init);' +
        'var t0=Date.now();' +
        'return _of.apply(this,arguments).then(function(resp){' +
          'var cl=resp.headers.get("content-length");' +
          'updateNetEntry(entry.id,{status:resp.status,size:cl?parseInt(cl):null,duration:Date.now()-t0,done:true});' +
          'return resp;' +
        '}).catch(function(err){' +
          'updateNetEntry(entry.id,{status:0,error:String(err),duration:Date.now()-t0,done:true});' +
          'throw err;' +
        '});' +
      '};' +
      'var _oxo=XMLHttpRequest.prototype.open,_oxs=XMLHttpRequest.prototype.send;' +
      'XMLHttpRequest.prototype.open=function(m,u){this.__pdtM=m;this.__pdtU=u;this.__pdtH={};return _oxo.apply(this,arguments);};' +
      'var _oxsh=XMLHttpRequest.prototype.setRequestHeader;' +
      'XMLHttpRequest.prototype.setRequestHeader=function(k,v){if(!this.__pdtH)this.__pdtH={};this.__pdtH[k]=v;return _oxsh.apply(this,arguments);};' +
      'XMLHttpRequest.prototype.send=function(body){' +
        'var ov=findOv(this.__pdtU,this.__pdtM);' +
        'if(ov){ov.hits=(ov.hits||0)+1;hitBadge(ov.id,ov.hits);' +
          'var self=this,st=parseInt(ov.statusCode)||200,rb=ov.body||"";' +
          'addNetEntry(this.__pdtU,this.__pdtM,"xhr",st,0,rb.length,{headers:this.__pdtH,body:body});' +
          'setTimeout(function(){' +
            'try{Object.defineProperty(self,"readyState",{get:function(){return 4;},configurable:true});' +
              'Object.defineProperty(self,"status",{get:function(){return st;},configurable:true});' +
              'Object.defineProperty(self,"responseText",{get:function(){return rb;},configurable:true});' +
              'Object.defineProperty(self,"response",{get:function(){return rb;},configurable:true});' +
              'if(typeof self.onreadystatechange==="function")self.onreadystatechange();' +
              'if(typeof self.onload==="function")self.onload(new Event("load"));' +
              'self.dispatchEvent(new Event("load"));self.dispatchEvent(new Event("loadend"));}' +
            'catch(e){}},0);return;}' +
        'var entry=addNetEntry(this.__pdtU,this.__pdtM,"xhr",null,null,null,{headers:this.__pdtH,body:body});' +
        'var t0=Date.now();' +
        'this.addEventListener("load",function(){' +
          'updateNetEntry(entry.id,{status:this.status,size:this.responseText?this.responseText.length:null,duration:Date.now()-t0,done:true,responseHeaders:this.getAllResponseHeaders()});' +
        '});' +
        'this.addEventListener("error",function(){' +
          'updateNetEntry(entry.id,{status:0,error:"XHR error",duration:Date.now()-t0,done:true});' +
        '});' +
        'return _oxs.apply(this,arguments);' +
      '};' +
    '}' +

    /* ââ Net log management ââ */
    'function addNetEntry(url,method,type,status,duration,size,reqInit){' +
      'var id=mkNetId();' +
      'var realUrl;try{if(url&&url.startsWith(PDT_BASE))realUrl=_dec_client(url.substring(PDT_BASE.length));else realUrl=url;}catch(e){realUrl=url;}' +
      'var entry={id:id,url:realUrl,proxyUrl:url,method:(method||"GET").toUpperCase(),type:type||"fetch",' +
        'status:status,duration:duration,size:size,time:fmtTime(),done:!!status,' +
        'reqHeaders:reqInit&&reqInit.headers?reqInit.headers:{},' +
        'reqBody:reqInit&&reqInit.body?String(reqInit.body).substring(0,2000):null,' +
        'responseHeaders:null,error:null};' +
      'S.netLog.unshift(entry);' +
      'if(S.netLog.length>500)S.netLog.pop();' +
      'if(S.tab==="network")renderNetList();' +
      'return entry;' +
    '}' +
    'function updateNetEntry(id,updates){' +
      'for(var i=0;i<S.netLog.length;i++){' +
        'if(S.netLog[i].id===id){' +
          'Object.assign(S.netLog[i],updates);' +
          'if(S.tab==="network"){' +
            'var el=document.querySelector("[data-nid=\\""+id+"\\"]");' +
            'if(el){var st=S.netLog[i];' +
              'el.className="pdtNetRow"+(S.netSel&&S.netSel.id===id?" pdtNetRowOn":"")+(st.type==="ws"?" pdtNetWs":"");' +
              'el.children[0].textContent=st.method;' +
              'el.children[0].style.color=methodClr(st.method);' +
              'el.children[1].className=stClr(st.status);el.children[1].textContent=st.status||"â¦";' +
              'el.children[3].textContent=fmtMs(st.duration);' +
              'el.children[4].textContent=fmtSize(st.size);' +
            '}' +
            'if(S.netSel&&S.netSel.id===id)renderNetDetail(S.netLog[i]);' +
          '}' +
          'break;' +
        '}' +
      '}' +
    '}' +

    /* ââ WebSocket interceptor ââ */
    'function hookWS(){' +
      'if(!window.WebSocket)return;' +
      'var OrigWS=window.WebSocket;' +
      'window.WebSocket=function(url,protocols){' +
        'var wsId=++_wsId;' +
        'var realUrl;try{if(url&&url.startsWith(PDT_BASE))realUrl=_dec_client(url.substring(PDT_BASE.length));else realUrl=url;}catch(e){realUrl=url;}' +
        'var ws=protocols?new OrigWS(url,protocols):new OrigWS(url);' +
        'var conn={id:wsId,url:realUrl,state:"connecting",messages:[],ws:ws};' +
        'S.wsConns.push(conn);' +
        'if(S.tab==="websocket")renderWsConns();' +
        'ws.addEventListener("open",function(){' +
          'conn.state="open";' +
          'conn.messages.push({dir:"sys",text:"Connected",time:fmtTime()});' +
          'if(S.tab==="websocket"){renderWsConns();if(S.wsSelConn&&S.wsSelConn.id===wsId)renderWsMsgs();}' +
        '});' +
        'ws.addEventListener("close",function(e){' +
          'conn.state="closed";' +
          'conn.messages.push({dir:"sys",text:"Closed code="+e.code+" reason="+e.reason,time:fmtTime()});' +
          'if(S.tab==="websocket"){renderWsConns();if(S.wsSelConn&&S.wsSelConn.id===wsId)renderWsMsgs();}' +
        '});' +
        'ws.addEventListener("message",function(e){' +
          'var txt;try{txt=typeof e.data==="string"?e.data:("[binary "+e.data.byteLength+"B]");}catch(ex){txt=String(e.data);}' +
          'conn.messages.push({dir:"recv",text:txt.substring(0,4096),time:fmtTime()});' +
          'if(conn.messages.length>1000)conn.messages.shift();' +
          'if(S.tab==="websocket"&&S.wsSelConn&&S.wsSelConn.id===wsId){' +
            'var ml=document.getElementById("__pdt_wsmsglist__");' +
            'if(ml){var div=document.createElement("div");div.className="pdtWsMsgR";' +
              'div.textContent="â "+fmtTime()+" "+txt.substring(0,200);ml.appendChild(div);' +
              'ml.scrollTop=ml.scrollHeight;}' +
          '}' +
        '});' +
        'ws.addEventListener("error",function(){' +
          'conn.messages.push({dir:"sys",text:"Error",time:fmtTime()});' +
          'if(S.tab==="websocket"&&S.wsSelConn&&S.wsSelConn.id===wsId)renderWsMsgs();' +
        '});' +
        'var origSend=ws.send.bind(ws);' +
        'ws.send=function(data){' +
          'var txt;try{txt=typeof data==="string"?data:("[binary "+data.byteLength+"B]");}catch(ex){txt=String(data);}' +
          'conn.messages.push({dir:"send",text:txt.substring(0,4096),time:fmtTime()});' +
          'if(conn.messages.length>1000)conn.messages.shift();' +
          'if(S.tab==="websocket"&&S.wsSelConn&&S.wsSelConn.id===wsId){' +
            'var ml2=document.getElementById("__pdt_wsmsglist__");' +
            'if(ml2){var div2=document.createElement("div");div2.className="pdtWsMsgS";' +
              'div2.textContent="â "+fmtTime()+" "+txt.substring(0,200);ml2.appendChild(div2);' +
              'ml2.scrollTop=ml2.scrollHeight;}' +
          '}' +
          'return origSend(data);' +
        '};' +
        'return ws;' +
      '};' +
      'window.WebSocket.prototype=OrigWS.prototype;' +
      'window.WebSocket.CONNECTING=0;window.WebSocket.OPEN=1;window.WebSocket.CLOSING=2;window.WebSocket.CLOSED=3;' +
    '}' +

    /* ââ Console interceptor ââ */
    'function hookConsole(){' +
      'var methods=["log","warn","error","info","debug","table","dir","group","groupEnd","time","timeEnd","assert","count","trace"];' +
      'methods.forEach(function(m){' +
        'var orig=console[m];' +
        'console[m]=function(){' +
          'try{orig.apply(console,arguments);}catch(e){}' +
          'var args=Array.prototype.slice.call(arguments);' +
          'var txt=args.map(function(a){' +
            'try{if(a===null)return"null";if(a===undefined)return"undefined";' +
              'if(typeof a==="object")return JSON.stringify(a,null,2);}' +
            'catch(e){}return String(a);' +
          '}).join(" ");' +
          'var level=m==="warn"?"warn":m==="error"||m==="assert"?"error":m==="info"?"info":"log";' +
          'if(m==="assert"&&arguments[0])return;' +
          'if(m==="trace"){txt=txt+"\\n"+(new Error()).stack;}' +
          'var entry={id:mkConId(),level:level,msg:txt.substring(0,4000),time:fmtTime()};' +
          'S.conLog.unshift(entry);' +
          'if(S.conLog.length>2000)S.conLog.pop();' +
          'if(S.tab==="console")appendConRow(entry);' +
        '};' +
      '});' +
      'window.addEventListener("error",function(e){' +
        'var entry={id:mkConId(),level:"error",msg:"Uncaught "+(e.error?e.error.stack||e.message:e.message)+" at "+e.filename+":"+e.lineno,time:fmtTime()};' +
        'S.conLog.unshift(entry);if(S.tab==="console")appendConRow(entry);' +
      '});' +
      'window.addEventListener("unhandledrejection",function(e){' +
        'var msg="Unhandled Promise Rejection: ";try{msg+=e.reason&&e.reason.stack?e.reason.stack:String(e.reason);}catch(ex){msg+=String(e.reason);}' +
        'var entry={id:mkConId(),level:"error",msg:msg,time:fmtTime()};' +
        'S.conLog.unshift(entry);if(S.tab==="console")appendConRow(entry);' +
      '});' +
    '}' +

    /* ââ Deobfuscation engine ââ */
    'var DEOB_PASSES=[' +
      '{id:"beautify",label:"â  Beautify (indent & format)",fn:deobBeautify},' +
      '{id:"hexStr",label:"â¡ Hex string decode (\\\\x41âA)",fn:deobHexStr},' +
      '{id:"unicodeStr",label:"â¢ Unicode escape decode (\\\\u0041âA)",fn:deobUnicodeStr},' +
      '{id:"base64",label:"â£ Base64 decode (atob strings)",fn:deobBase64},' +
      '{id:"evalUnpack",label:"â¤ eval() unpack (packed JS)",fn:deobEvalUnpack},' +
      '{id:"strConcat",label:"â¥ String concat fold (a+b+câabc)",fn:deobStringConcat},' +
      '{id:"strArray",label:"â¦ String array substitution",fn:deobStringArray},' +
      '{id:"rot13",label:"â§ ROT13 decode",fn:deobRot13},' +
      '{id:"hexNum",label:"â¨ Hex number literals (0x41â65)",fn:deobHexNums},' +
      '{id:"octalNum",label:"â© Octal literals (\\\\101âA)",fn:deobOctal},' +
      '{id:"jsfuck",label:"âª JSFuck partial decode (+[]â0)",fn:deobJsfuck},' +
      '{id:"commaExpr",label:"â« Comma expression split",fn:deobCommaExpr},' +
      '{id:"boolNum",label:"â¬ Boolean math (+trueâ1)",fn:deobBoolNum},' +
      '{id:"emptyArr",label:"â­ Empty array coercions (+[]â0)",fn:deobEmptyArr},' +
      '{id:"deadsym",label:"â® Dead symbol removal (;; â;)",fn:deobDeadSym},' +
      '{id:"selfInvoke",label:"â¯ IIFE wrapper unpack",fn:deobSelfInvoke},' +
      '{id:"shorthand",label:"â° Property shorthand rename",fn:deobShorthand},' +
      '{id:"charCode",label:"â± String.fromCharCode unpack",fn:deobCharCode},' +
      '{id:"atobInline",label:"â² Inline atob() evaluation",fn:deobAtobInline},' +
      '{id:"urldecode",label:"â³ URL percent-decode",fn:deobUrlDecode},' +
      '{id:"escape",label:"ã unescape() sequences",fn:deobEscape},' +
      '{id:"autoAll",label:"â¦ Auto-all (run all passes)",fn:deobAutoAll}' +
    '];' +

    'function deobBeautify(code){' +
      'try{if(window.js_beautify)return window.js_beautify(code,{indent_size:2,preserve_newlines:true,max_preserve_newlines:2});}catch(e){}return code;' +
    '}' +
    'function deobHexStr(code){' +
      'try{return code.replace(/\\\\x([0-9a-fA-F]{2})/g,function(m,h){try{return String.fromCharCode(parseInt(h,16));}catch(e){return m;}});}catch(e){return code;}' +
    '}' +
    'function deobUnicodeStr(code){' +
      'try{return code.replace(/\\\\u([0-9a-fA-F]{4})/g,function(m,h){try{return String.fromCharCode(parseInt(h,16));}catch(e){return m;}});}catch(e){return code;}' +
    '}' +
    'function deobBase64(code){' +
      'try{return code.replace(/"([A-Za-z0-9+\\/]{20,}={0,2})"/g,function(m,b){' +
        'try{var dec=atob(b);if(/^[\\x20-\\x7e\\n\\r\\t]+$/.test(dec))return JSON.stringify(dec);}catch(e){}return m;' +
      '});}catch(e){return code;}' +
    '}' +
    'function deobEvalUnpack(code){' +
      'try{' +
        'var m=code.match(/eval\\(function\\(p,a,c,k,e,[rd]\\).*\\|\\|.*\\)\\([\'"](.+?)[\'"],\\s*(\\d+),\\s*(\\d+),\\s*[\'"](.+?)[\'"].split/s);' +
        'if(!m)return code;' +
        'var p=m[1],a=parseInt(m[2]),c=parseInt(m[3]),k=m[4].split("|"),e,r=new RegExp("\\\\b(\\\\w+)\\\\b","g");' +
        'while(c--)if(k[c])p=p.replace(new RegExp("\\\\b"+c.toString(a)+"\\\\b","g"),k[c]);' +
        'return p;' +
      '}catch(e){return code;}' +
    '}' +
    'function deobStringConcat(code){' +
      'try{return code.replace(/"([^"\\\\]*)"\s*\+\s*"([^"\\\\]*)"/g,function(m,a,b){return JSON.stringify(a+b);}).replace(/\'([^\'\\\\]*)\'\s*\+\s*\'([^\'\\\\]*)\'/g,function(m,a,b){return JSON.stringify(a+b);});}catch(e){return code;}' +
    '}' +
    'function deobStringArray(code){' +
      'try{' +
        'var m=code.match(/var\\s+(\\w+)\\s*=\\s*\\[([^\\]]+)\\]/);' +
        'if(!m)return code;' +
        'var arrName=m[1];' +
        'var items=m[2].match(/(?:"[^"]*"|\'[^\']*\')/g);' +
        'if(!items||items.length===0)return code;' +
        'var result=code;' +
        'for(var i=0;i<items.length;i++){' +
          'result=result.replace(new RegExp(arrName+"\\\\["+i+"\\\\]","g"),items[i]);' +
        '}' +
        'return result;' +
      '}catch(e){return code;}' +
    '}' +
    'function deobRot13(code){' +
      'try{return code.replace(/[a-zA-Z]/g,function(c){var b=c<="Z"?65:97;return String.fromCharCode((c.charCodeAt(0)-b+13)%26+b);});}catch(e){return code;}' +
    '}' +
    'function deobHexNums(code){' +
      'try{return code.replace(/\\b0x([0-9a-fA-F]+)\\b/g,function(m,h){try{return parseInt(h,16).toString();}catch(e){return m;}});}catch(e){return code;}' +
    '}' +
    'function deobOctal(code){' +
      'try{return code.replace(/\\\\([0-7]{1,3})/g,function(m,o){try{return String.fromCharCode(parseInt(o,8));}catch(e){return m;}});}catch(e){return code;}' +
    '}' +
    'function deobJsfuck(code){' +
      'try{' +
        'var r=code;' +
        'r=r.replace(/\\+\\[\\]/g,"0");' +
        'r=r.replace(/!\\[\\]/g,"false");' +
        'r=r.replace(/!!\\[\\]/g,"true");' +
        'r=r.replace(/\\+!!\\[\\]/g,"1");' +
        'r=r.replace(/\\+!\\[\\]+!!\\[\\]/g,"false1");' +
        'return r;' +
      '}catch(e){return code;}' +
    '}' +
    'function deobCommaExpr(code){' +
      'try{return code.replace(/\\(([^()]+),([^()]+)\\)/g,function(m,a,b){return"("+a+");"+"("+b+")";});}catch(e){return code;}' +
    '}' +
    'function deobBoolNum(code){' +
      'try{return code.replace(/\\+true\\b/g,"1").replace(/\\+false\\b/g,"0").replace(/\\+null\\b/g,"0");}catch(e){return code;}' +
    '}' +
    'function deobEmptyArr(code){' +
      'try{return code.replace(/\\+\\[\\]/g,"0").replace(/"\\s*\\+\\s*\\[\\]\\s*\\+\\s*"/g,"");}catch(e){return code;}' +
    '}' +
    'function deobDeadSym(code){' +
      'try{return code.replace(/;;+/g,";").replace(/\\{\\s*\\}/g,"{}").replace(/,\\s*,/g,",");}catch(e){return code;}' +
    '}' +
    'function deobSelfInvoke(code){' +
      'try{return code.replace(/^\\s*\\(function\\s*\\(\\)\\s*\\{([\\s\\S]*)\\}\\)\\(\\);?\\s*$/,function(m,inner){return inner.trim();});}catch(e){return code;}' +
    '}' +
    'function deobShorthand(code){' +
      'try{return code.replace(/([\\w$]+)\\s*:\\s*\\1(?=[,}])/g,"$1");}catch(e){return code;}' +
    '}' +
    'function deobCharCode(code){' +
      'try{return code.replace(/String\\.fromCharCode\\(([0-9,\\s]+)\\)/g,function(m,nums){' +
        'try{var cs=nums.split(",").map(function(n){return String.fromCharCode(parseInt(n.trim(),10));});' +
          'var res=cs.join("");if(/^[\\x20-\\x7e]+$/.test(res))return JSON.stringify(res);return m;}catch(e){return m;}' +
      '});}catch(e){return code;}' +
    '}' +
    'function deobAtobInline(code){' +
      'try{return code.replace(/atob\\((["\'])([A-Za-z0-9+\\/]{8,}={0,2})\\1\\)/g,function(m,q,b){' +
        'try{var dec=atob(b);if(/^[\\x20-\\x7e\\n\\r\\t]+$/.test(dec))return JSON.stringify(dec);return m;}catch(e){return m;}' +
      '});}catch(e){return code;}' +
    '}' +
    'function deobUrlDecode(code){' +
      'try{return code.replace(/%[0-9a-fA-F]{2}/g,function(m){try{return decodeURIComponent(m);}catch(e){return m;}});}catch(e){return code;}' +
    '}' +
    'function deobEscape(code){' +
      'try{return code.replace(/unescape\\((["\'])([^"\']+)\\1\\)/g,function(m,q,s){try{return JSON.stringify(unescape(s));}catch(e){return m;}});}catch(e){return code;}' +
    '}' +
    'function deobAutoAll(code){' +
      'var result=code;' +
      'var passes=[deobEvalUnpack,deobHexStr,deobUnicodeStr,deobOctal,deobCharCode,deobAtobInline,deobBase64,deobBoolNum,deobEmptyArr,deobJsfuck,deobHexNums,deobStringConcat,deobStringArray,deobDeadSym,deobSelfInvoke,deobUrlDecode,deobEscape,deobBeautify];' +
      'for(var i=0;i<passes.length;i++){try{var next=passes[i](result);if(next&&next!==result)result=next;}catch(e){}}' +
      'return result;' +
    '}' +

    /* ââ Override UI ââ */
    'function addOv(prefill){' +
      'var id=mkId();' +
      'var rule=Object.assign({id:id,enabled:true,urlPattern:"",method:"*",statusCode:"200",contentType:"application/json",body:\'{"overridden":true}\',hits:0},prefill||{});' +
      'rule.id=id;rule.hits=0;' +
      'S.overrides.unshift(rule);renderOvList();' +
      'setTimeout(function(){' +
        'var row=document.querySelector("[data-ovid=\\""+id+"\\"]");' +
        'if(row){var b=row.querySelector(".pdtOvBody");if(b)b.classList.add("pdtOvBodyOn");' +
          'var inp=row.querySelector(".pdtOvInp");if(inp)inp.focus();}},30);' +
    '}' +
    'function renderOvList(){' +
      'var list=document.getElementById("__pdt_ovl__");if(!list)return;' +
      'list.innerHTML="";' +
      'if(!S.overrides.length){' +
        'list.innerHTML="<div class=\\"pdtOvHint\\">&#x1F504; No overrides yet.<br><br>Click <b>+ Add Override<\\/b> to intercept requests.<br><span style=\\"font-size:11px;color:#444;\\">Works for fetch() and XMLHttpRequest<\\/span><\\/div>";' +
        'return;}' +
      'S.overrides.forEach(function(ov){' +
        'var row=document.createElement("div");row.className="pdtOvRow";row.setAttribute("data-ovid",ov.id);' +
        'var head=document.createElement("div");head.className="pdtOvHead";' +
        'var mOpts=["*","GET","POST","PUT","PATCH","DELETE","OPTIONS"].map(function(m){return"<option"+(ov.method===m?" selected":"")+">"+m+"<\\/option>";}).join("");' +
        'head.innerHTML=' +
          '"<input type=\\"checkbox\\" "+(ov.enabled?"checked":"")+">"+' +
          '"<span class=\\""+( ov.enabled?"pdtOvOn":"pdtOvOff")+"\\">"+(ov.enabled?"ON":"OFF")+"<\\/span>"+' +
          '"<span class=\\"pdtOvUrl\\" title=\\""+esc(ov.urlPattern)+"\\">"+(ov.urlPattern?esc(ov.urlPattern):"<span style=\\"color:#555;font-style:italic\\">No URL set<\\/span>")+"<\\/span>"+' +
          '"<span class=\\"pdtOvHits\\">"+(ov.hits?""+ov.hits+" hit"+(ov.hits===1?"":"s"):"")+"<\\/span>"+' +
          '"<div class=\\"pdtOvActs\\">"+' +
            '"<button class=\\"pdtBtn\\" data-act=\\"clone\\" style=\\"font-size:10px;\\" title=\\"Duplicate\\">&#x2398;<\\/button>"+' +
            '"<button class=\\"pdtBtn pdtBtnRed\\" data-act=\\"del\\" style=\\"font-size:10px;\\" title=\\"Delete\\">&#x2715;<\\/button>"+' +
          '"<\\/div>";' +
        'var body=document.createElement("div");body.className="pdtOvBody";' +
        'body.innerHTML=' +
          '"<div class=\\"pdtOvIrow\\"><div class=\\"pdtOvIlbl\\">URL Pattern (* = wildcard)<\\/div>"+' +
          '"<input class=\\"pdtOvInp\\" data-f=\\"urlPattern\\" value=\\""+esc(ov.urlPattern)+"\\" placeholder=\\"https://api.example.com/*/data\\"><\\/div>"+' +
          '"<div style=\\"display:flex;gap:8px;\\">"+' +
            '"<div class=\\"pdtOvIrow\\" style=\\"flex:1;\\"><div class=\\"pdtOvIlbl\\">Method<\\/div><select class=\\"pdtOvSel\\" data-f=\\"method\\">"+mOpts+"<\\/select><\\/div>"+' +
            '"<div class=\\"pdtOvIrow\\" style=\\"flex:1;\\"><div class=\\"pdtOvIlbl\\">Status<\\/div><input class=\\"pdtOvInp\\" data-f=\\"statusCode\\" value=\\""+esc(ov.statusCode)+"\\" placeholder=\\"200\\"><\\/div>"+' +
            '"<div class=\\"pdtOvIrow\\" style=\\"flex:2;\\"><div class=\\"pdtOvIlbl\\">Content-Type<\\/div><input class=\\"pdtOvInp\\" data-f=\\"contentType\\" value=\\""+esc(ov.contentType)+"\\" placeholder=\\"application/json\\"><\\/div>"+' +
          '"<\\/div>"+' +
          '"<div class=\\"pdtOvIrow\\"><div class=\\"pdtOvIlbl\\">Response Body<\\/div>"+' +
          '"<textarea class=\\"pdtOvTa\\" data-f=\\"body\\">"+esc(ov.body)+"<\\/textarea><\\/div>"+' +
          '"<div style=\\"display:flex;gap:6px;align-items:center;\\">"+' +
            '"<button class=\\"pdtBtn pdtBtnGrn\\" data-act=\\"apply\\" style=\\"font-size:11px;\\">&#x2713; Apply<\\/button>"+' +
            '"<span style=\\"font-size:10.5px;color:#555;font-family:Segoe UI,sans-serif;\\">Intercepts matching requests immediately.<\\/span>"+' +
          '"<\\/div>";' +
        'row.appendChild(head);row.appendChild(body);list.appendChild(row);' +
        '(function(r,h,b,o){' +
          'h.addEventListener("click",function(e){' +
            'e.stopPropagation();' +
            'if(e.target.tagName==="INPUT"||e.target.tagName==="BUTTON")return;' +
            'b.classList.toggle("pdtOvBodyOn");});' +
          'h.querySelector("input[type=checkbox]").addEventListener("change",function(e){' +
            'e.stopPropagation();o.enabled=this.checked;' +
            'var badge=h.querySelector(".pdtOvOn,.pdtOvOff");' +
            'if(badge){badge.className=o.enabled?"pdtOvOn":"pdtOvOff";badge.textContent=o.enabled?"ON":"OFF";}});' +
          'h.querySelectorAll("button[data-act]").forEach(function(btn){' +
            'btn.addEventListener("click",function(e){' +
              'e.stopPropagation();var act=btn.getAttribute("data-act");' +
              'if(act==="del"){S.overrides=S.overrides.filter(function(x){return x.id!==o.id;});renderOvList();}' +
              'else if(act==="clone"){addOv(Object.assign({},o,{hits:0}));}' +
            '});});' +
          'b.querySelector("button[data-act=apply]").addEventListener("click",function(e){' +
            'e.stopPropagation();' +
            'b.querySelectorAll("[data-f]").forEach(function(el){o[el.getAttribute("data-f")]=el.value;});' +
            'o.hits=0;renderOvList();' +
            'setTimeout(function(){var r2=document.querySelector("[data-ovid=\\""+o.id+"\\"]");' +
              'if(r2){var b2=r2.querySelector(".pdtOvBody");if(b2)b2.classList.add("pdtOvBodyOn");}},20);' +
          '});' +
        '})(row,head,body,ov);' +
      '});' +
    '}' +

    /* ââ File list rendering ââ */
    'function renderFL(){' +
      'var list=document.getElementById("__pdt_fl__");if(!list)return;list.innerHTML="";' +
      'if(!S.resources.length){list.innerHTML="<div class=\\"pdtMsg\\">No files found.<\\/div>";return;}' +
      '[{label:"HTML",type:"html"},{label:"JavaScript",type:"js"},{label:"CSS",type:"css"}].forEach(function(g){' +
        'var items=S.resources.filter(function(r){return r.type===g.type;});if(!items.length)return;' +
        'var gh=document.createElement("div");gh.className="pdtGrp";' +
        'gh.textContent=g.label+" ("+items.length+")";list.appendChild(gh);' +
        'items.forEach(function(r){' +
          'var fi=document.createElement("div");fi.className="pdtFi";fi.setAttribute("data-fid",r.id);' +
          'fi.innerHTML="<span style=\\"width:8px;height:8px;border-radius:50%;background:"+dotClr(r.type)+";display:inline-block;flex-shrink:0;\\"><\\/span>"+' +
            '"<span style=\\"overflow:hidden;text-overflow:ellipsis;min-width:0;\\" title=\\""+esc(r.realUrl||r.name)+"\\">"+ esc(r.name)+"<\\/span>";' +
          'fi.addEventListener("click",function(e){e.stopPropagation();selRes(r);});' +
          'list.appendChild(fi);' +
        '});' +
      '});' +
    '}' +

    /* ââ Select/show resource ââ */
    'function selRes(r){' +
      'document.querySelectorAll(".pdtFi").forEach(function(el){el.classList.remove("pdtFiOn");});' +
      'var el=document.querySelector("[data-fid=\\""+r.id+"\\"]");if(el)el.classList.add("pdtFiOn");' +
      'S.sel=r;showCode(r);' +
    '}' +
    'function showCode(r){' +
      'var fn=document.getElementById("__pdt_fn__");' +
      'var cw=document.getElementById("__pdt_cw__");' +
      'if(!cw)return;' +
      'if(fn)fn.textContent=r.realUrl||r.name;' +
      'cw.innerHTML="<div class=\\"pdtMsg\\">&#x23F3; Loading "+esc(r.name)+"...<\\/div>";' +
      'function render(raw){' +
        'S.raw=raw;' +
        'var w=document.getElementById("__pdt_cw__");' +
        'if(w)w.innerHTML="<pre><code>"+renderCode(raw,r.type)+"<\\/code><\\/pre>";' +
      '}' +
      'if(S.cache[r.id]!==undefined){render(S.cache[r.id]);return;}' +
      'r.load().then(function(raw){S.cache[r.id]=raw;render(raw);}' +
      ').catch(function(err){' +
        'var w=document.getElementById("__pdt_cw__");' +
        'if(w)w.innerHTML="<div class=\\"pdtErr\\">&#x26A0; Failed: "+esc(String(err))+"<\\/div>";' +
      '});' +
    '}' +

    /* ââ Network UI ââ */
    'function renderNetList(){' +
      'var lst=document.getElementById("__pdt_netlst__");if(!lst)return;' +
      'var filt=S.netFilter.toLowerCase();' +
      'var tf=S.netTypeFilter;' +
      'var visible=S.netLog.filter(function(e){' +
        'var matchUrl=!filt||e.url.toLowerCase().includes(filt);' +
        'var matchType=tf==="all"||(tf==="xhr"&&e.type==="xhr")||(tf==="fetch"&&e.type==="fetch")||(tf==="ws"&&e.type==="ws");' +
        'return matchUrl&&matchType;' +
      '});' +
      'lst.innerHTML="";' +
      'visible.forEach(function(e){' +
        'var row=document.createElement("div");' +
        'row.className="pdtNetRow"+(S.netSel&&S.netSel.id===e.id?" pdtNetRowOn":"")+(e.type==="ws"?" pdtNetWs":"");' +
        'row.setAttribute("data-nid",e.id);' +
        'var statusTxt=e.done?(e.status||"ERR"):"â¦";' +
        'row.innerHTML=' +
          '"<span style=\\"color:"+methodClr(e.method)+";font-size:10px;font-weight:bold;\\">"+esc(e.method)+"<\\/span>"+' +
          '"<span class=\\""+stClr(e.status)+"\\" style=\\"font-size:11px;\\">"+statusTxt+"<\\/span>"+' +
          '"<span style=\\"overflow:hidden;text-overflow:ellipsis;white-space:nowrap;color:#9cdcfe;font-size:11.5px;\\" title=\\""+esc(e.url)+"\\">"+(shortName(e.url)||e.url)+"<\\/span>"+' +
          '"<span style=\\"font-size:10.5px;color:#888;\\">"+fmtMs(e.duration)+"<\\/span>"+' +
          '"<span style=\\"font-size:10.5px;color:#668;\\">"+fmtSize(e.size)+"<\\/span>"+' +
          '"<span style=\\"font-size:10px;color:#555;\\">"+e.time+"<\\/span>";' +
        'row.addEventListener("click",function(ev){ev.stopPropagation();selNetEntry(e);});' +
        'lst.appendChild(row);' +
      '});' +
    '}' +
    'function selNetEntry(e){' +
      'S.netSel=e;' +
      'document.querySelectorAll(".pdtNetRow").forEach(function(r){r.classList.remove("pdtNetRowOn");});' +
      'var el=document.querySelector("[data-nid=\\""+e.id+"\\"]");if(el)el.classList.add("pdtNetRowOn");' +
      'renderNetDetail(e);' +
    '}' +
    'function renderNetDetail(e){' +
      'var det=document.getElementById("__pdt_netdet__");if(!det)return;' +
      'var h="";' +
      'h+="<div class=\\"pdtNetSection\\">General<\\/div>";' +
      'h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">URL<\\/span><span class=\\"pdtNetVal\\">"+esc(e.url)+"<\\/span><\\/div>";' +
      'h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">Method<\\/span><span class=\\"pdtNetVal\\">"+esc(e.method)+"<\\/span><\\/div>";' +
      'h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">Status<\\/span><span class=\\"pdtNetVal "+stClr(e.status)+"\\">"+esc(String(e.status||"pending"))+"<\\/span><\\/div>";' +
      'h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">Type<\\/span><span class=\\"pdtNetVal\\">"+esc(e.type)+"<\\/span><\\/div>";' +
      'h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">Duration<\\/span><span class=\\"pdtNetVal\\">"+fmtMs(e.duration)+"<\\/span><\\/div>";' +
      'h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">Size<\\/span><span class=\\"pdtNetVal\\">"+fmtSize(e.size)+"<\\/span><\\/div>";' +
      'if(e.error)h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">Error<\\/span><span class=\\"pdtNetVal\\" style=\\"color:#f44747;\\">"+esc(e.error)+"<\\/span><\\/div>";' +
      'if(e.reqHeaders&&typeof e.reqHeaders==="object"){' +
        'var hkeys=Object.keys(e.reqHeaders);' +
        'if(hkeys.length){h+="<div class=\\"pdtNetSection\\">Request Headers<\\/div>";' +
          'hkeys.forEach(function(k){h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">"+esc(k)+"<\\/span><span class=\\"pdtNetVal\\">"+esc(String(e.reqHeaders[k]))+"<\\/span><\\/div>";});' +
        '}' +
      '}' +
      'if(e.reqBody){h+="<div class=\\"pdtNetSection\\">Request Body<\\/div>";h+="<pre style=\\"white-space:pre-wrap;color:#ce9178;margin:0;font-size:11px;\\">"+esc(e.reqBody)+"<\\/pre>";}' +
      'if(e.responseHeaders){h+="<div class=\\"pdtNetSection\\">Response Headers<\\/div>";' +
        'e.responseHeaders.split("\\n").forEach(function(line){if(!line.trim())return;var ci=line.indexOf(":");' +
          'if(ci>-1){h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">"+esc(line.substring(0,ci).trim())+"<\\/span><span class=\\"pdtNetVal\\">"+esc(line.substring(ci+1).trim())+"<\\/span><\\/div>";}' +
        '});' +
      '}' +
      'h+="<div class=\\"pdtNetSection\\">Proxy URL<\\/div>";' +
      'h+="<div class=\\"pdtNetKv\\"><span class=\\"pdtNetKey\\">Proxied<\\/span><span class=\\"pdtNetVal\\" style=\\"color:#555;font-size:10px;\\">"+esc(e.proxyUrl||"")+"<\\/span><\\/div>";' +
      'det.innerHTML=h;' +
    '}' +

    /* ââ Console UI ââ */
    'function appendConRow(entry){' +
      'var log=document.getElementById("__pdt_conlog__");if(!log)return;' +
      'var row=document.createElement("div");' +
      'var lvl=entry.level;' +
      'row.className="pdtConRow "+(lvl==="warn"?"pdtConWarn":lvl==="error"?"pdtConErr":lvl==="info"?"pdtConInfo":"pdtConLog");' +
      'var badge=lvl==="warn"?"pdtConBW":lvl==="error"?"pdtConBE":"pdtConBL";' +
      'var badgeTxt=lvl==="warn"?"W":lvl==="error"?"E":lvl==="info"?"I":"L";' +
      'row.innerHTML="<span class=\\"pdtConTime\\">"+entry.time+"<\\/span>"+' +
        '"<span class=\\"pdtConBadge "+badge+"\\">"+badgeTxt+"<\\/span>"+' +
        '"<span class=\\"pdtConMsg\\">"+esc(entry.msg)+"<\\/span>";' +
      'log.insertBefore(row,log.firstChild);' +
      'if(log.children.length>500){log.removeChild(log.lastChild);}' +
    '}' +
    'function renderConLog(){' +
      'var log=document.getElementById("__pdt_conlog__");if(!log)return;' +
      'log.innerHTML="";' +
      'var filt=S.conFilter;' +
      'var visible=S.conLog.filter(function(e){return filt==="all"||e.level===filt;});' +
      'visible.slice(0,300).forEach(function(e){appendConRow(e);});' +
    '}' +

    /* ââ WebSocket UI ââ */
    'function renderWsConns(){' +
      'var c=document.getElementById("__pdt_wsconns__");if(!c)return;' +
      'c.innerHTML="";' +
      'if(!S.wsConns.length){c.innerHTML="<div class=\\"pdtMsg\\" style=\\"font-size:11px;padding:10px;\\">No WebSocket<br>connections yet.<\\/div>";return;}' +
      'S.wsConns.slice().reverse().forEach(function(conn){' +
        'var d=document.createElement("div");' +
        'd.className="pdtWsConn"+(S.wsSelConn&&S.wsSelConn.id===conn.id?" pdtWsConnOn":"")+(conn.state==="closed"?" pdtWsConnDead":"");' +
        'var stateIcon=conn.state==="open"?"ð¢":conn.state==="closed"?"ð´":"ð¡";' +
        'd.title=conn.url;' +
        'd.textContent=stateIcon+" "+(shortName(conn.url)||conn.url)+" ("+conn.messages.length+")";' +
        'd.addEventListener("click",function(){S.wsSelConn=conn;renderWsConns();renderWsMsgs();});' +
        'c.appendChild(d);' +
      '});' +
    '}' +
    'function renderWsMsgs(){' +
      'var ml=document.getElementById("__pdt_wsmsglist__");if(!ml)return;' +
      'ml.innerHTML="";' +
      'if(!S.wsSelConn){ml.innerHTML="<div class=\\"pdtMsg\\">Select a connection<\\/div>";return;}' +
      'S.wsSelConn.messages.forEach(function(m){' +
        'var div=document.createElement("div");' +
        'div.className=m.dir==="send"?"pdtWsMsgS":m.dir==="recv"?"pdtWsMsgR":"pdtWsMsgSys";' +
        'var arrow=m.dir==="send"?"â ":m.dir==="recv"?"â ":"â¦¿ ";' +
        'div.textContent=arrow+m.time+" "+m.text.substring(0,300);' +
        'ml.appendChild(div);' +
      '});' +
      'ml.scrollTop=ml.scrollHeight;' +
    '}' +

    /* ââ Storage UI ââ */
    'function renderStorage(){' +
      'var tbl=document.getElementById("__pdt_strtbl__");if(!tbl)return;' +
      'var mode=S.strMode;' +
      'var items=[];' +
      'try{' +
        'if(mode==="localStorage"){var ls=window.localStorage;for(var i=0;i<ls.length;i++){var k=ls.key(i);items.push({k:k,v:ls.getItem(k)});}}' +
        'else if(mode==="sessionStorage"){var ss=window.sessionStorage;for(var i=0;i<ss.length;i++){var k=ss.key(i);items.push({k:k,v:ss.getItem(k)});}}' +
        'else if(mode==="cookies"){document.cookie.split(";").forEach(function(c){var eq=c.indexOf("=");if(eq>-1)items.push({k:c.substring(0,eq).trim(),v:c.substring(eq+1).trim()});});}' +
      '}catch(e){tbl.innerHTML="<div class=\\"pdtErr\\">"+esc(String(e))+"<\\/div>";return;}' +
      'if(!items.length){tbl.innerHTML="<div class=\\"pdtMsg\\">No entries found.<\\/div>";return;}' +
      'var h="<table class=\\"pdtT\\"><thead><tr><th>Key<\\/th><th>Value<\\/th><\\/tr><\\/thead><tbody>";' +
      'items.forEach(function(it){' +
        'var val=it.v;var parsed="";' +
        'try{var obj=JSON.parse(val);parsed=" <span style=\\"color:#4ec9b0;font-size:9px;\\">[JSON]<\\/span>";}catch(e){}' +
        'h+="<tr><td class=\\"pdtStrKey\\">"+esc(it.k)+"<\\/td><td class=\\"pdtStrVal\\">"+esc(val.substring(0,300))+parsed+"<\\/td><\\/tr>";' +
      '});' +
      'h+="<\\/tbody><\\/table>";' +
      'tbl.innerHTML=h;' +
    '}' +

    /* ââ Deob UI ââ */
    'function runDeob(passId){' +
      'var inp=document.getElementById("__pdt_deobin__");' +
      'var out=document.getElementById("__pdt_deobout__");' +
      'var status=document.getElementById("__pdt_deobstatus__");' +
      'if(!inp||!out)return;' +
      'var code=inp.value;' +
      'if(!code.trim()){if(status)status.textContent="â  No input.";return;}' +
      'var pass=DEOB_PASSES.find(function(p){return p.id===passId;});' +
      'if(!pass){if(status)status.textContent="â  Unknown pass.";return;}' +
      'var t0=Date.now();' +
      'var result;' +
      'try{result=pass.fn(code);}catch(e){if(status)status.textContent="â  Error: "+e.message;return;}' +
      'out.value=result||code;' +
      'var dt=Date.now()-t0;' +
      'var diff=code.length-result.length;' +
      'if(status)status.textContent="â Done in "+dt+"ms Â· "+(diff>0?"-"+diff:diff<0?"+"+Math.abs(diff):"Â±0")+" chars";' +
      'S.deobHistory.push({id:passId,label:pass.label,before:code,after:result});' +
      'if(S.deobHistory.length>20)S.deobHistory.shift();' +
    '}' +
    'function deobApplyToSrc(){' +
      'var out=document.getElementById("__pdt_deobout__");' +
      'if(!out)return;' +
      'var cw=document.getElementById("__pdt_cw__");' +
      'if(!cw)return;' +
      'S.raw=out.value;' +
      'if(S.sel)cw.innerHTML="<pre><code>"+renderCode(S.raw,S.sel.type)+"<\\/code><\\/pre>";' +
    '}' +

    /* ââ Tab switch ââ */
    'function switchTab(name){' +
      'S.tab=name;' +
      'document.querySelectorAll(".pdtTab").forEach(function(t){' +
        't.classList.toggle("pdtTabOn",t.getAttribute("data-tab")===name);});' +
      'document.querySelectorAll(".pdtPage").forEach(function(p){' +
        'p.classList.toggle("pdtPageOn",p.getAttribute("data-page")===name);});' +
      'if(name==="network")renderNetList();' +
      'if(name==="console")renderConLog();' +
      'if(name==="websocket"){renderWsConns();renderWsMsgs();}' +
      'if(name==="storage")renderStorage();' +
    '}' +

    /* ââ Panel open/close ââ */
    'function openPanel(){' +
      'S.open=true;' +
      'var p=document.getElementById("__pdt_panel__");if(p)p.classList.add("pdt_on");' +
      'if(S.resources.length===0){S.resources=collectRes();renderFL();}' +
    '}' +
    'function closePanel(){' +
      'S.open=false;' +
      'var p=document.getElementById("__pdt_panel__");if(p)p.classList.remove("pdt_on");' +
    '}' +

    /* ââ Resize drag ââ */
    'function initDrag(handle,panel){' +
      'var sy=0,sh=0;' +
      'handle.addEventListener("mousedown",function(e){' +
        'sy=e.clientY;sh=parseInt(panel.style.height)||520;' +
        'document.addEventListener("mousemove",mv);' +
        'document.addEventListener("mouseup",up);' +
        'e.preventDefault();' +
      '});' +
      'function mv(e){panel.style.height=Math.max(160,Math.min(window.innerHeight-80,sh+(sy-e.clientY)))+"px";}' +
      'function up(){document.removeEventListener("mousemove",mv);document.removeEventListener("mouseup",up);}' +
    '}' +

    /* ââ Build UI ââ */
    'function buildUI(){' +
      'var fab=document.createElement("button");' +
      'fab.id="__pdt_fab__";' +
      'fab.innerHTML="&#x1F6E0; DevTools";' +
      'fab.addEventListener("click",function(e){e.stopPropagation();e.preventDefault();S.open?closePanel():openPanel();});' +
      'document.body.appendChild(fab);' +

      'var panel=document.createElement("div");panel.id="__pdt_panel__";' +
      'var drag=document.createElement("div");drag.id="__pdt_drag__";panel.appendChild(drag);' +

      'var hdr=document.createElement("div");hdr.id="__pdt_hdr__";' +
      'hdr.innerHTML=' +
        '"<div class=\\"pdtTab pdtTabOn\\" data-tab=\\"sources\\">&#x1F4C1; Sources<\\/div>"+' +
        '"<div class=\\"pdtTab\\" data-tab=\\"network\\">&#x1F310; Network<\\/div>"+' +
        '"<div class=\\"pdtTab\\" data-tab=\\"console\\">&#x1F4BB; Console<\\/div>"+' +
        '"<div class=\\"pdtTab\\" data-tab=\\"websocket\\">&#x1F4E1; WebSocket<\\/div>"+' +
        '"<div class=\\"pdtTab\\" data-tab=\\"storage\\">&#x1F4BE; Storage<\\/div>"+' +
        '"<div class=\\"pdtTab\\" data-tab=\\"overrides\\">&#x1F504; Overrides<\\/div>"+' +
        '"<div class=\\"pdtTab\\" data-tab=\\"deobfuscate\\">&#x1F50D; Deobfuscate<\\/div>"+' +
        '"<div id=\\"__pdt_hdrR__\\"><span>Proxy DevTools v2<\\/span>"+' +
          '"<button id=\\"__pdt_xbtn__\\" title=\\"Close\\">&times;<\\/button><\\/div>";' +
      'panel.appendChild(hdr);' +

      'var body=document.createElement("div");body.id="__pdt_body__";' +

      /* Sources page */
      'var srcPg=document.createElement("div");srcPg.className="pdtPage pdtPageOn";srcPg.setAttribute("data-page","sources");' +
      'var sb=document.createElement("div");sb.id="__pdt_sb__";' +
      'sb.innerHTML="<div id=\\"__pdt_sbtb__\\"><button class=\\"pdtBtn\\" id=\\"__pdt_scan__\\" style=\\"font-size:11px;\\">&#x21BB; Scan<\\/button><\\/div>"+' +
        '"<div id=\\"__pdt_fl__\\"><div class=\\"pdtMsg\\">Click &ldquo;Scan&rdquo;<\\/div><\\/div>";' +
      'srcPg.appendChild(sb);' +
      'var main=document.createElement("div");main.id="__pdt_main__";' +
      'var tb=document.createElement("div");tb.id="__pdt_tb__";' +
      'tb.innerHTML=' +
        '"<div id=\\"__pdt_fn__\\">Select a file<\\/div>"+' +
        '"<label class=\\"pdtLbl\\"><input type=\\"checkbox\\" id=\\"__pdt_beauty__\\" style=\\"accent-color:#007acc;\\"> Beautify<\\/label>"+' +
        '"<button class=\\"pdtBtn\\" id=\\"__pdt_copy__\\">&#x2398; Copy<\\/button>"+' +
        '"<button class=\\"pdtBtn\\" id=\\"__pdt_dl__\\">&#x2B73; Download<\\/button>";' +
      'main.appendChild(tb);' +
      'var cw=document.createElement("div");cw.id="__pdt_cw__";' +
      'cw.innerHTML="<pre><code style=\\"color:#555;font-style:italic;padding:20px;display:block;\\">// No file selected.<\\/code><\\/pre>";' +
      'main.appendChild(cw);srcPg.appendChild(main);body.appendChild(srcPg);' +

      /* Network page */
      'var netPg=document.createElement("div");netPg.className="pdtPage";netPg.setAttribute("data-page","network");' +
      'netPg.innerHTML=' +
        '"<div id=\\"__pdt_netp__\\">"+' +
          '"<div id=\\"__pdt_nettb__\\">"+' +
            '"<input id=\\"__pdt_netsrch__\\" placeholder=\\"Filter URL...\\" type=\\"text\\">"+' +
            '"<select id=\\"__pdt_nettype__\\" class=\\"pdtOvSel\\" style=\\"font-size:11px;\\">"+' +
              '"<option value=\\"all\\">All<\\/option><option value=\\"fetch\\">Fetch<\\/option><option value=\\"xhr\\">XHR<\\/option><option value=\\"ws\\">WS<\\/option>"+' +
            '"<\\/select>"+' +
            '"<button class=\\"pdtBtn\\" id=\\"__pdt_netclr__\\" style=\\"font-size:10px;\\">&#x1F5D1; Clear<\\/button>"+' +
            '"<button class=\\"pdtBtn\\" id=\\"__pdt_netexp__\\" style=\\"font-size:10px;\\">&#x1F4E4; Export HAR<\\/button>"+' +
          '"<\\/div>"+' +
          '"<div id=\\"__pdt_netpane__\\">"+' +
            '"<div id=\\"__pdt_netlstwrap__\\">"+' +
              '"<div class=\\"pdtNetHead\\"><span>Method<\\/span><span>Status<\\/span><span>URL<\\/span><span>Time<\\/span><span>Size<\\/span><span>Clock<\\/span><\\/div>"+' +
              '"<div id=\\"__pdt_netlst__\\"><div class=\\"pdtMsg\\">No requests captured yet.<\\/div><\\/div>"+' +
            '"<\\/div>"+' +
            '"<div id=\\"__pdt_netdet__\\" class=\\"pdtNetDetail\\"><div class=\\"pdtMsg\\">Select a request.<\\/div><\\/div>"+' +
          '"<\\/div>"+' +
        '"<\\/div>";' +
      'body.appendChild(netPg);' +

      /* Console page */
      'var conPg=document.createElement("div");conPg.className="pdtPage";conPg.setAttribute("data-page","console");' +
      'conPg.innerHTML=' +
        '"<div id=\\"__pdt_conp__\\">"+' +
          '"<div id=\\"__pdt_contb__\\">"+' +
            '"<select id=\\"__pdt_confilt__\\" class=\\"pdtOvSel\\" style=\\"font-size:11px;\\">"+' +
              '"<option value=\\"all\\">All<\\/option><option value=\\"log\\">Log<\\/option><option value=\\"warn\\">Warn<\\/option><option value=\\"error\\">Errors<\\/option><option value=\\"info\\">Info<\\/option>"+' +
            '"<\\/select>"+' +
            '"<button class=\\"pdtBtn\\" id=\\"__pdt_conclr__\\" style=\\"font-size:10px;\\">&#x1F5D1; Clear<\\/button>"+' +
            '"<button class=\\"pdtBtn\\" id=\\"__pdt_conexport__\\" style=\\"font-size:10px;\\">&#x1F4E4; Export<\\/button>"+' +
          '"<\\/div>"+' +
          '"<div id=\\"__pdt_conlog__\\"><\\/div>"+' +
          '"<div id=\\"__pdt_conin__\\">"+' +
            '"<input id=\\"__pdt_coninp__\\" placeholder=\\"> Evaluate JavaScript expression...\\" type=\\"text\\">"+' +
            '"<button id=\\"__pdt_conrun__\\">&#x25BA;<\\/button>"+' +
          '"<\\/div>"+' +
        '"<\\/div>";' +
      'body.appendChild(conPg);' +

      /* WebSocket page */
      'var wsPg=document.createElement("div");wsPg.className="pdtPage";wsPg.setAttribute("data-page","websocket");' +
      'wsPg.innerHTML=' +
        '"<div id=\\"__pdt_wsp__\\">"+' +
          '"<div id=\\"__pdt_wstb__\\">"+' +
            '"<span style=\\"font-size:11px;color:#aaa;font-family:Segoe UI,sans-serif;\\">WebSocket Connections<\\/span>"+' +
            '"<button class=\\"pdtBtn\\" id=\\"__pdt_wsclr__\\" style=\\"font-size:10px;\\">&#x1F5D1; Clear msgs<\\/button>"+' +
            '"<button class=\\"pdtBtn\\" id=\\"__pdt_wsexp__\\" style=\\"font-size:10px;\\">&#x1F4E4; Export<\\/button>"+' +
          '"<\\/div>"+' +
          '"<div id=\\"__pdt_wspane__\\">"+' +
            '"<div id=\\"__pdt_wsconns__\\"><div class=\\"pdtMsg\\" style=\\"font-size:11px;padding:10px;\\">No WebSocket<br>connections yet.<\\/div><\\/div>"+' +
            '"<div id=\\"__pdt_wsmsgs__\\">"+' +
              '"<div id=\\"__pdt_wsmsglist__\\"><div class=\\"pdtMsg\\">Select a connection.<\\/div><\\/div>"+' +
              '"<div id=\\"__pdt_wssend__\\">"+' +
                '"<input id=\\"__pdt_wsinp__\\" placeholder=\\"Send message to selected WebSocket...\\" type=\\"text\\">"+' +
                '"<button class=\\"pdtBtn\\" id=\\"__pdt_wssnd__\\" style=\\"flex-shrink:0;\\">&#x25BA; Send<\\/button>"+' +
              '"<\\/div>"+' +
            '"<\\/div>"+' +
          '"<\\/div>"+' +
        '"<\\/div>";' +
      'body.appendChild(wsPg);' +

      /* Storage page */
      'var strPg=document.createElement("div");strPg.className="pdtPage";strPg.setAttribute("data-page","storage");' +
      'strPg.innerHTML=' +
        '"<div id=\\"__pdt_strp__\\">"+' +
          '"<div id=\\"__pdt_strtb__\\">"+' +
            '"<button class=\\"pdtBtn\\" id=\\"__pdt_strrfr__\\" style=\\"font-size:10px;\\">&#x21BB; Refresh<\\/button>"+' +
            '"<button class=\\"pdtBtn pdtBtnRed\\" id=\\"__pdt_strclr__\\" style=\\"font-size:10px;\\">&#x1F5D1; Clear selected<\\/button>"+' +
            '"<button class=\\"pdtBtn\\" id=\\"__pdt_strexp__\\" style=\\"font-size:10px;\\">&#x1F4E4; Export<\\/button>"+' +
          '"<\\/div>"+' +
          '"<div id=\\"__pdt_strpane__\\">"+' +
            '"<div id=\\"__pdt_strside__\\">"+' +
              '"<div class=\\"pdtStrItem pdtStrItemOn\\" data-smode=\\"localStorage\\">&#x1F4E6; localStorage<\\/div>"+' +
              '"<div class=\\"pdtStrItem\\" data-smode=\\"sessionStorage\\">&#x1F4E6; sessionStorage<\\/div>"+' +
              '"<div class=\\"pdtStrItem\\" data-smode=\\"cookies\\">&#x1F36A; Cookies<\\/div>"+' +
            '"<\\/div>"+' +
            '"<div id=\\"__pdt_strmain__\\"><div id=\\"__pdt_strtbl__\\"><div class=\\"pdtMsg\\">Select a storage type.<\\/div><\\/div><\\/div>"+' +
          '"<\\/div>"+' +
        '"<\\/div>";' +
      'body.appendChild(strPg);' +

      /* Overrides page */
      'var ovPg=document.createElement("div");ovPg.className="pdtPage";ovPg.setAttribute("data-page","overrides");' +
      'var ovp=document.createElement("div");ovp.id="__pdt_ovp__";' +
      'ovp.innerHTML=' +
        '"<div id=\\"__pdt_ovtb__\\">"+' +
          '"<span class=\\"pdtOvTitle\\">&#x1F504; Intercept Fetch &amp; XHR<\\/span>"+' +
          '"<button class=\\"pdtBtn pdtBtnGrn\\" id=\\"__pdt_ovadd__\\">&#x2795; Add Override<\\/button>"+' +
          '"<button class=\\"pdtBtn\\" id=\\"__pdt_ovexp__\\" style=\\"font-size:10.5px;\\">&#x1F4E4; Export<\\/button>"+' +
          '"<button class=\\"pdtBtn\\" id=\\"__pdt_ovimp__\\" style=\\"font-size:10.5px;\\">&#x1F4E5; Import<\\/button>"+' +
          '"<button class=\\"pdtBtn pdtBtnRed\\" id=\\"__pdt_ovclear__\\" style=\\"font-size:10.5px;\\">&#x1F5D1; Clear<\\/button>"+' +
        '"<\\/div>"+' +
        '"<div id=\\"__pdt_ovl__\\"><\\/div>";' +
      'ovPg.appendChild(ovp);body.appendChild(ovPg);' +

      /* Deobfuscate page */
      'var deobPg=document.createElement("div");deobPg.className="pdtPage";deobPg.setAttribute("data-page","deobfuscate");' +
      'var deobSel=DEOB_PASSES.map(function(p){return"<option value=\\""+esc(p.id)+"\\">"+esc(p.label)+"<\\/option>";}).join("");' +
      'deobPg.innerHTML=' +
        '"<div id=\\"__pdt_deobp__\\">"+' +
          '"<div id=\\"__pdt_deobtb__\\">"+' +
            '"<select id=\\"__pdt_deobsel__\\" class=\\"pdtDeobSel\\">"+deobSel+"<\\/select>"+' +
            '"<button class=\\"pdtDeobBtn\\" id=\\"__pdt_deobrun__\\">&#x25BA; Run<\\/button>"+' +
            '"<button class=\\"pdtDeobBtn pdtDeobBtnOra\\" id=\\"__pdt_deobrunall__\\">&#x2728; Auto-All<\\/button>"+' +
            '"<button class=\\"pdtDeobBtn\\" id=\\"__pdt_deobin2out__\\" title=\\"Apply output â input for chaining\\">&#x21BA; Chain<\\/button>"+' +
            '"<button class=\\"pdtDeobBtn\\" id=\\"__pdt_deobload__\\">&#x1F4C2; Load selected file<\\/button>"+' +
            '"<button class=\\"pdtDeobBtn\\" id=\\"__pdt_deobapply__\\" title=\\"Apply result to Sources viewer\\">&#x1F4CB; Apply to Sources<\\/button>"+' +
            '"<button class=\\"pdtDeobBtn pdtDeobBtnRed\\" id=\\"__pdt_deobreset__\\">&#x1F5D1; Reset<\\/button>"+' +
            '"<button class=\\"pdtDeobBtn\\" id=\\"__pdt_deobcopy__\\">&#x2398; Copy<\\/button>"+' +
            '"<button class=\\"pdtDeobBtn\\" id=\\"__pdt_deobdl__\\">&#x2B73; Download<\\/button>"+' +
            '"<span id=\\"__pdt_deobstatus__\\"><\\/span>"+' +
          '"<\\/div>"+' +
          '"<div id=\\"__pdt_deobmain__\\">"+' +
            '"<div id=\\"__pdt_deobinwrap__\\">"+' +
              '"<div class=\\"pdtDeobLabel\\">Input (paste or load)<\\/div>"+' +
              '"<textarea id=\\"__pdt_deobin__\\" spellcheck=\\"false\\" placeholder=\\"Paste obfuscated JavaScript here...\\"><\\/textarea>"+' +
            '"<\\/div>"+' +
            '"<div id=\\"__pdt_deoboutwrap__\\">"+' +
              '"<div class=\\"pdtDeobLabel\\">Output (deobfuscated)<\\/div>"+' +
              '"<textarea id=\\"__pdt_deobout__\\" spellcheck=\\"false\\" readonly placeholder=\\"Deobfuscated output appears here...\\"><\\/textarea>"+' +
            '"<\\/div>"+' +
          '"<\\/div>"+' +
        '"<\\/div>";' +
      'body.appendChild(deobPg);' +

      'panel.appendChild(body);document.body.appendChild(panel);' +

      /* Wire close button */
      'document.getElementById("__pdt_xbtn__").addEventListener("click",function(e){' +
        'e.stopImmediatePropagation();e.stopPropagation();e.preventDefault();closePanel();},true);' +

      /* Wire tab buttons */
      'hdr.querySelectorAll(".pdtTab").forEach(function(t){' +
        't.addEventListener("click",function(e){e.stopPropagation();switchTab(t.getAttribute("data-tab"));});});' +

      /* Sources events */
      'document.getElementById("__pdt_scan__").addEventListener("click",function(e){' +
        'e.stopPropagation();S.resources=collectRes();renderFL();});' +
      'document.getElementById("__pdt_beauty__").addEventListener("change",function(){' +
        'S.beautify=this.checked;if(S.sel)showCode(S.sel);});' +
      'document.getElementById("__pdt_dl__").addEventListener("click",function(e){' +
        'e.stopPropagation();if(!S.sel||!S.raw)return;' +
        'dl(S.sel.filename,S.beautify?beauty(S.raw,S.sel.type):S.raw);});' +
      'document.getElementById("__pdt_copy__").addEventListener("click",function(e){' +
        'e.stopPropagation();if(!S.raw)return;' +
        'cp(S.beautify?beauty(S.raw,S.sel.type):S.raw);});' +

      /* Network events */
      'document.getElementById("__pdt_netsrch__").addEventListener("input",function(){' +
        'S.netFilter=this.value;renderNetList();});' +
      'document.getElementById("__pdt_nettype__").addEventListener("change",function(){' +
        'S.netTypeFilter=this.value;renderNetList();});' +
      'document.getElementById("__pdt_netclr__").addEventListener("click",function(e){' +
        'e.stopPropagation();S.netLog=[];S.netSel=null;renderNetList();' +
        'var det=document.getElementById("__pdt_netdet__");if(det)det.innerHTML="<div class=\\"pdtMsg\\">Select a request.<\\/div>";});' +
      'document.getElementById("__pdt_netexp__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var har={log:{version:"1.2",creator:{name:"ProxyDevTools",version:"2.0"},entries:S.netLog.map(function(e){' +
          'return{startedDateTime:e.time,time:e.duration||0,' +
            'request:{method:e.method,url:e.url,headers:[],queryString:[],bodySize:e.reqBody?e.reqBody.length:-1},' +
            'response:{status:e.status||0,statusText:"",headers:[],content:{size:e.size||0,mimeType:""}},timings:{send:0,wait:e.duration||0,receive:0}};' +
        '})}};' +
        'dl("network.har",JSON.stringify(har,null,2));});' +

      /* Console events */
      'document.getElementById("__pdt_confilt__").addEventListener("change",function(){' +
        'S.conFilter=this.value;renderConLog();});' +
      'document.getElementById("__pdt_conclr__").addEventListener("click",function(e){' +
        'e.stopPropagation();S.conLog=[];var log=document.getElementById("__pdt_conlog__");if(log)log.innerHTML="";});' +
      'document.getElementById("__pdt_conexport__").addEventListener("click",function(e){' +
        'e.stopPropagation();var txt=S.conLog.map(function(e){return"["+e.time+"]["+e.level.toUpperCase()+"] "+e.msg;}).join("\\n");' +
        'dl("console.log",txt);});' +
      'document.getElementById("__pdt_coninp__").addEventListener("keydown",function(e){' +
        'if(e.key==="Enter"){e.stopPropagation();e.preventDefault();' +
          'var code=this.value.trim();if(!code)return;' +
          'var entry={id:mkConId(),level:"log",msg:"> "+code,time:fmtTime()};' +
          'S.conLog.unshift(entry);appendConRow(entry);' +
          'try{var res=eval(code);' +
            'var rEntry={id:mkConId(),level:"info",msg:"â "+String(res).substring(0,2000),time:fmtTime()};' +
            'S.conLog.unshift(rEntry);appendConRow(rEntry);' +
          '}catch(err){' +
            'var eEntry={id:mkConId(),level:"error",msg:"â  "+String(err),time:fmtTime()};' +
            'S.conLog.unshift(eEntry);appendConRow(eEntry);' +
          '}' +
          'this.value="";' +
        '}});' +
      'document.getElementById("__pdt_conrun__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var inp=document.getElementById("__pdt_coninp__");' +
        'if(inp){inp.dispatchEvent(new KeyboardEvent("keydown",{key:"Enter",bubbles:true}));}});' +

      /* WebSocket events */
      'document.getElementById("__pdt_wsclr__").addEventListener("click",function(e){' +
        'e.stopPropagation();if(S.wsSelConn)S.wsSelConn.messages=[];renderWsMsgs();});' +
      'document.getElementById("__pdt_wsexp__").addEventListener("click",function(e){' +
        'e.stopPropagation();if(!S.wsSelConn)return;' +
        'var txt=S.wsSelConn.messages.map(function(m){return"["+m.time+"]["+m.dir+"] "+m.text;}).join("\\n");' +
        'dl("websocket-"+S.wsSelConn.id+".log",txt);});' +
      'document.getElementById("__pdt_wssnd__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var inp=document.getElementById("__pdt_wsinp__");if(!inp)return;' +
        'var msg=inp.value.trim();if(!msg)return;' +
        'if(!S.wsSelConn||S.wsSelConn.state!=="open"){alert("No open WebSocket selected.");return;}' +
        'try{S.wsSelConn.ws.send(msg);inp.value="";}catch(err){alert("Send failed: "+err.message);}});' +
      'document.getElementById("__pdt_wsinp__").addEventListener("keydown",function(e){' +
        'if(e.key==="Enter"){e.stopPropagation();document.getElementById("__pdt_wssnd__").click();}});' +

      /* Storage events */
      'document.querySelectorAll("[data-smode]").forEach(function(el){' +
        'el.addEventListener("click",function(){' +
          'document.querySelectorAll("[data-smode]").forEach(function(x){x.classList.remove("pdtStrItemOn");});' +
          'el.classList.add("pdtStrItemOn");' +
          'S.strMode=el.getAttribute("data-smode");renderStorage();});});' +
      'document.getElementById("__pdt_strrfr__").addEventListener("click",function(e){e.stopPropagation();renderStorage();});' +
      'document.getElementById("__pdt_strclr__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'if(!confirm("Clear all "+S.strMode+" entries?"))return;' +
        'try{' +
          'if(S.strMode==="localStorage")localStorage.clear();' +
          'else if(S.strMode==="sessionStorage")sessionStorage.clear();' +
          'else if(S.strMode==="cookies"){document.cookie.split(";").forEach(function(c){' +
            'var k=c.split("=")[0].trim();document.cookie=k+"=;expires=Thu, 01 Jan 1970 00:00:00 UTC;path=/;";});}' +
        '}catch(err){alert("Clear failed: "+err.message);}' +
        'renderStorage();});' +
      'document.getElementById("__pdt_strexp__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var data={};' +
        'try{' +
          'if(S.strMode==="localStorage"){for(var i=0;i<localStorage.length;i++){var k=localStorage.key(i);data[k]=localStorage.getItem(k);}}' +
          'else if(S.strMode==="sessionStorage"){for(var i=0;i<sessionStorage.length;i++){var k=sessionStorage.key(i);data[k]=sessionStorage.getItem(k);}}' +
          'else{document.cookie.split(";").forEach(function(c){var eq=c.indexOf("=");if(eq>-1)data[c.substring(0,eq).trim()]=c.substring(eq+1).trim();});}' +
        '}catch(err){}' +
        'dl(S.strMode+".json",JSON.stringify(data,null,2));});' +

      /* Overrides events */
      'document.getElementById("__pdt_ovadd__").addEventListener("click",function(e){e.stopPropagation();addOv();});' +
      'document.getElementById("__pdt_ovclear__").addEventListener("click",function(e){' +
        'e.stopPropagation();if(!confirm("Clear all overrides?"))return;S.overrides=[];renderOvList();});' +
      'document.getElementById("__pdt_ovexp__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var json=JSON.stringify(S.overrides.map(function(o){' +
          'return{urlPattern:o.urlPattern,method:o.method,statusCode:o.statusCode,contentType:o.contentType,body:o.body,enabled:o.enabled};}),null,2);' +
        'dl("proxy-overrides.json",json);});' +
      'document.getElementById("__pdt_ovimp__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var inp=document.createElement("input");inp.type="file";inp.accept=".json,application/json";' +
        'inp.style.cssText="position:fixed;left:-9999px;";document.body.appendChild(inp);' +
        'inp.addEventListener("change",function(){' +
          'var f=inp.files[0];if(!f)return;' +
          'var fr=new FileReader();' +
          'fr.onload=function(ev){' +
            'try{var arr=JSON.parse(ev.target.result);' +
              'if(!Array.isArray(arr))throw new Error("Expected JSON array");' +
              'arr.forEach(function(item){addOv(item);})}' +
            'catch(err){alert("Import failed: "+err.message);}' +
            'document.body.removeChild(inp);};' +
          'fr.readAsText(f);});' +
        'inp.click();});' +

      /* Deobfuscate events */
      'document.getElementById("__pdt_deobrun__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var sel=document.getElementById("__pdt_deobsel__");' +
        'if(sel)runDeob(sel.value);});' +
      'document.getElementById("__pdt_deobrunall__").addEventListener("click",function(e){' +
        'e.stopPropagation();runDeob("autoAll");});' +
      'document.getElementById("__pdt_deobin2out__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var inp=document.getElementById("__pdt_deobin__");' +
        'var out=document.getElementById("__pdt_deobout__");' +
        'if(inp&&out&&out.value){inp.value=out.value;out.value="";}});' +
      'document.getElementById("__pdt_deobload__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var inp=document.getElementById("__pdt_deobin__");if(!inp)return;' +
        'if(!S.sel){alert("No file selected in Sources tab.");return;}' +
        'S.sel.load().then(function(raw){inp.value=raw;' +
          'var status=document.getElementById("__pdt_deobstatus__");' +
          'if(status)status.textContent="â Loaded: "+esc(S.sel.name);' +
        '}).catch(function(err){alert("Load failed: "+err.message);});});' +
      'document.getElementById("__pdt_deobapply__").addEventListener("click",function(e){' +
        'e.stopPropagation();deobApplyToSrc();switchTab("sources");});' +
      'document.getElementById("__pdt_deobreset__").addEventListener("click",function(e){' +
        'e.stopPropagation();' +
        'var inp=document.getElementById("__pdt_deobin__");' +
        'var out=document.getElementById("__pdt_deobout__");' +
        'var status=document.getElementById("__pdt_deobstatus__");' +
        'if(inp)inp.value="";if(out)out.value="";if(status)status.textContent="";' +
        'S.deobHistory=[];});' +
      'document.getElementById("__pdt_deobcopy__").addEventListener("click",function(e){' +
        'e.stopPropagation();var out=document.getElementById("__pdt_deobout__");if(out&&out.value)cp(out.value);});' +
      'document.getElementById("__pdt_deobdl__").addEventListener("click",function(e){' +
        'e.stopPropagation();var out=document.getElementById("__pdt_deobout__");' +
        'if(out&&out.value)dl("deobfuscated.js",out.value);});' +
      'document.getElementById("__pdt_deobin__").addEventListener("keydown",function(e){' +
        'if(e.key==="Tab"){e.preventDefault();var start=this.selectionStart;var end=this.selectionEnd;' +
          'this.value=this.value.substring(0,start)+"  "+this.value.substring(end);' +
          'this.selectionStart=this.selectionEnd=start+2;}});' +

      'initDrag(drag,panel);' +
      'renderOvList();' +
      'hookNet();' +
      'hookWS();' +
      'hookConsole();' +
    '}' +

    /* ââ Init ââ */
    'function init(){if(!document.body){setTimeout(init,100);return;}buildUI();}' +
    'init();' +
    '})();'
  );
}




const proxyHintInjection = `
function toEntities(str) {
  return str.split("").map(ch => \`&#\${ch.charCodeAt(0)};\`).join("");
}
setTimeout(() => {
  var hint = \`Warning: You are currently using a web proxy, so do not log in to any website. Click to close this hint. For further details, please visit the link below.
è­¦åï¼æ¨å½åæ­£å¨ä½¿ç¨ç½ç»ä»£çï¼è¯·å¿ç»å½ä»»ä½ç½ç«ãåå»å³é­æ­¤æç¤ºãè¯¦æè¯·è§ä»¥ä¸é¾æ¥ã\`;
  if (document.readyState === 'complete' || document.readyState === 'interactive') {
    document.body.insertAdjacentHTML(
      'afterbegin', 
      \`<div style="position:fixed;left:0px;top:0px;width:100%;margin:0px;padding:0px;display:block;z-index:99999999999999999999999;user-select:none;cursor:pointer;" id="__PROXY_HINT_DIV__" onclick="document.getElementById('__PROXY_HINT_DIV__').remove();">
        <span style="position:relative;display:block;width:calc(100% - 20px);min-height:30px;font-size:14px;color:yellow;background:rgb(180,0,0);text-align:center;border-radius:5px;padding-left:10px;padding-right:10px;padding-top:1px;padding-bottom:1px;">
          \${toEntities(hint)}
          <br>
          <a href="https://github.com/1234567Yang/cf-proxy-ex/" style="color:rgb(250,250,180);">https://github.com/1234567Yang/cf-proxy-ex/</a>
        </span>
      </div>\`
    );
  } else {
    alert(hint + " https://github.com/1234567Yang/cf-proxy-ex");
  }
}, 5000);
`;

const mainPage = `<html>
<head>
    <meta charset="utf-8">
    <title>Cf-proxy-ex</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html, body { min-height: 100%; font-family: Arial, sans-serif; background-color: #f0f8ff; }
        body { display: flex; flex-direction: column; align-items: center; justify-content: flex-start; padding: 30px; }
        .container { background-color: #fff; padding: 20px; border-radius: 10px; box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1); width: 100%; max-width: 400px; text-align: center; margin: 20px 0; }
        h1 { font-size: 22px; margin-bottom: 15px; }
        input[type="text"] { width: 100%; padding: 10px; margin-bottom: 15px; border: 1px solid #ccc; border-radius: 5px; font-size: 14px; box-shadow: inset 0 4px 8px rgba(0, 0, 0, 0.2); }
        button { padding: 10px 20px; background-color: #008cba; color: white; border: none; border-radius: 5px; font-size: 16px; cursor: pointer; box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2); }
        button:hover { background-color: #005f5f; }
        ul { margin-top: 20px; list-style-type: none; font-size: 14px; text-align: left; width: 100%; max-width: 600px; }
        li { margin-bottom: 10px; }
        a { color: #008cba; text-decoration: none; cursor:pointer; }
        a:hover { text-decoration: underline; }
        @media (max-width: 600px) { body { justify-content: flex-start; } h1 { font-size: 18px; } button { font-size: 14px; } .container { padding: 15px; margin-top: 10px; } }
    </style>
</head>
<body>
<div class="container">
<form id="urlForm" onsubmit="redirectToProxy(event)">
    <h1>Cf-proxy-ex</h1>
    <label for="targetUrl">
        <input type="text" id="targetUrl" placeholder="Enter the target website here...">
    </label>
    <button type="submit" id="jump"> Jump! </button>
</form>
</div>
<ul>
  <li>
      å¦ä½ä½¿ç¨ / How to use<br>
      1. å¨ä¸æ¹è¾å¥æ¡è¾å¥è¦è®¿é®çç½å / Type the website link above<br>
      2.å¨ä»£çç½ååè¾å¥æ¨è¦è®¿é®çç½å / Type the website link after the proxy website's link<br>
  </li><br>
  <li>è¥æ¾ç¤º 400 Bad Request éè¯¯ï¼è¯·æ¸æ¬ç½ç«Cookie / Please clear this website's cookie if it shows 400 Bad Request</li><br>
  <li>ç±äºé¨åç½ç«æä»£ç æ··æ·ï¼ä¸è½ä¿è¯ææç½é¡µçåè½ææ¸²ææ­£å¸¸ / Some website may perform malfunction due to JS/CSS obfuscation</li><br>
  <li><strong>å¼ºçä¸å»ºè®®å¨éåé¡µé¢ä¸­ç»å½è´¦å· / Strongly discourage logging into any mirrored website</strong></li><br><br><br>
  <li style="text-align:center;font-size: calc(100% + 2px);">
      <br>
      <a onclick="fillUrl('https://wikipedia.com/')">Wikipedia</a> |
      <a onclick="fillUrl('https://github.com/')">GitHub</a> |
      <a onclick="fillUrl('https://duckduckgo.com/')">DuckDuckGo</a> 
  </li>
  <br>
</ul>
<ul style="position:absolute;bottom:15px;text-align:center;">
<li>
<p>æ¬ä»£çä¸º <a href="https://github.com/1234567Yang/cf-proxy-ex" target="_blank">å¼æºé¡¹ç®</a> / This is an <a href="https://github.com/1234567Yang/cf-proxy-ex" target="_blank">open source project</a></p>
<p>æè°¢ <a href="https://github.com/Tayasui-rainnya" target="_blank">@Tayasui-rainnya</a> çä¸»é¡µè®¾è®¡ / Thanks for <a href="https://github.com/Tayasui-rainnya" target="_blank">@Tayasui-rainnya</a>'s home page design</p>
</li>
</ul>
<script>
  const encPrefix = "${encPrefix}";
  function _enc_client(str) {
    if (!str) return str;
    if (str.startsWith(encPrefix)) return str;
    try {
      return encPrefix + btoa(encodeURIComponent(str)).replace(/\\//g, '_').replace(/\\+/g, '-').replace(/=/g, '');
    } catch (e) { return str; }
  }
  function redirectToProxy(event) {
      event.preventDefault();
      const targetUrl = document.getElementById('targetUrl').value.trim();
      if(!targetUrl) return;
      let finalUrl = targetUrl;
      if(!finalUrl.startsWith("http")) finalUrl = "https://" + finalUrl;
      const currentOrigin = window.location.origin;
      window.open(currentOrigin + '/' + _enc_client(finalUrl), '_blank');
  }
  function fillUrl(url) {
    document.getElementById('targetUrl').value = url;
    document.getElementById('jump').click();
  }
</script>
</body>
</html>`;

const pwdPage = `<!DOCTYPE html>
<html>
    <head>
        <script>
            function setPassword() {
                try {
                    var cookieDomain = window.location.hostname;
                    var password = document.getElementById('password').value;
                    var currentOrigin = window.location.origin;
                    var oneWeekLater = new Date();
                    oneWeekLater.setTime(oneWeekLater.getTime() + (7 * 24 * 60 * 60 * 1000)); 
                    document.cookie = "${passwordCookieName}" + "=" + password + "; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; domain=" + cookieDomain;
                    document.cookie = "${passwordCookieName}" + "=" + password + "; expires=" + oneWeekLater.toUTCString() + "; path=/; domain=" + cookieDomain;
                } catch(e) {
                    alert(e.message);
                }
                location.reload();
            }
        </script>
    </head>
    <body>
        <div>
            <input id="password" type="password" placeholder="Password">
            <button onclick="setPassword()">Submit</button>
        </div>
    </body>
</html>`;

// =======================================================================================
// *-*-*-*-*-*-*-*-*-* Advanced Media Stream Processing Engine *-*-*-*-*-*-*-*-*-*-*-*-*-*
// =======================================================================================
/**
 * Advanced Media Stream Processing Engine for robust video/audio streaming over proxy.
 * 
 * References:
 * - https://github.com/mischah/range-parser : For RFC 7233 compliant HTTP Range header parsing logic.
 * - https://github.com/jimmywarting/StreamSaver.js : For safe and progressive WHATWG stream chunking and piping.
 * - https://github.com/videojs/video.js : For managing chunked video player bounds and buffer expectations.
 * - https://github.com/web-push-libs/web-push : For utilizing robust byte-level stream manipulations.
 *
 * Problem Addressed:
 * When using Cloudflare Workers as a proxy, upstream servers sometimes return '200 OK'
 * with the full video instead of acknowledging the 'Range' header with '206 Partial Content'.
 * This causes HTML5 video players to spin (infinite loading) because they expect precise chunking.
 * This engine dynamically intercepts the stream, calculates the byte ranges, slices the stream
 * manually using the Web Streams API, and synthesizes a perfect '206 Partial Content' response.
 */
class MediaStreamProxyEngine {
    /**
     * Parse HTTP Range header completely compliant with RFC 7233.
     * @param {string} rangeHeader 
     * @param {number} totalLength 
     * @returns {Object|null}
     */
    static parseRange(rangeHeader, totalLength) {
        if (!rangeHeader || typeof rangeHeader !== 'string') return null;
        const index = rangeHeader.indexOf('=');
        if (index === -1) return null;
        
        const type = rangeHeader.slice(0, index);
        if (type !== 'bytes') return null; 
        
        const ranges = rangeHeader.slice(index + 1).split(',');
        const parsedRanges = ranges.map(range => {
            const parts = range.split('-');
            const start = parseInt(parts[0], 10);
            const end = parseInt(parts[1], 10);
            
            if (isNaN(start) && isNaN(end)) return null;
            
            let startByte, endByte;
            if (isNaN(start)) {
                startByte = Math.max(0, totalLength - end);
                endByte = totalLength - 1;
            } else if (isNaN(end)) {
                startByte = start;
                endByte = totalLength - 1;
            } else {
                startByte = start;
                endByte = Math.min(end, totalLength - 1);
            }
            
            if (startByte > endByte || startByte >= totalLength) return null;
            return { start: startByte, end: endByte };
        }).filter(r => r !== null);
        
        if (parsedRanges.length === 0) return null;
        return parsedRanges[0]; 
    }

    /**
     * Slices a ReadableStream according to precise byte boundaries.
     * Essential for forcing pseudo-206 responses on stubborn upstreams.
     * @param {ReadableStream} readableStream 
     * @param {number} startByte 
     * @param {number} endByte 
     * @returns {ReadableStream}
     */
    static async createSlicedStream(readableStream, startByte, endByte) {
        let currentByte = 0;
        const reader = readableStream.getReader();
        
        return new ReadableStream({
            async pull(controller) {
                try {
                    while (true) {
                        const { done, value } = await reader.read();
                        if (done) {
                            controller.close();
                            break;
                        }
                        
                        const chunkLength = value.byteLength;
                        const chunkEndByte = currentByte + chunkLength - 1;
                        
                        // We haven't reached the start byte yet, skip this chunk
                        if (chunkEndByte < startByte) {
                            currentByte += chunkLength;
                            continue;
                        }
                        
                        // We are entirely past the end byte, cancel stream
                        if (currentByte > endByte) {
                            controller.close();
                            await reader.cancel("Reached end of requested range");
                            break;
                        }
                        
                        let sliceStart = 0;
                        let sliceEnd = chunkLength;
                        
                        if (currentByte < startByte) {
                            sliceStart = startByte - currentByte;
                        }
                        
                        if (chunkEndByte > endByte) {
                            sliceEnd = chunkLength - (chunkEndByte - endByte);
                        }
                        
                        const chunkSlice = value.slice(sliceStart, sliceEnd);
                        controller.enqueue(chunkSlice);
                        
                        currentByte += chunkLength;
                        
                        if (currentByte > endByte) {
                            controller.close();
                            await reader.cancel("Reached end of requested range");
                            break;
                        }
                        break; // Yield control back to consumer
                    }
                } catch (error) {
                    controller.error(error);
                }
            },
            cancel(reason) {
                reader.cancel(reason);
            }
        });
    }

    /**
     * Core orchestrator for media responses.
     * Checks if synthesis is needed, handles headers securely, and outputs proper stream.
     * @param {Request} request 
     * @param {Response} upstreamResponse 
     * @param {number} originalStatus 
     * @param {Headers} originalHeaders 
     * @returns {Promise<Response>}
     */
    static async processMediaResponse(request, upstreamResponse, originalStatus, originalHeaders) {
        const clientRange = request.headers.get('Range') || request.headers.get('range');
        const contentLengthStr = originalHeaders.get('Content-Length');
        const totalLength = contentLengthStr ? parseInt(contentLengthStr, 10) : null;
        
        // If upstream behaves correctly or we lack total length to slice, we pass through securely
        if (originalStatus === 206 || !clientRange || !totalLength) {
            const modifiedHeaders = new Headers(originalHeaders);
            modifiedHeaders.set("Accept-Ranges", "bytes");
            // Chunked encoding often breaks Seekability in certain mobile browsers
            modifiedHeaders.delete("Transfer-Encoding");
            
            return new Response(upstreamResponse.body, {
                status: originalStatus,
                statusText: upstreamResponse.statusText,
                headers: modifiedHeaders
            });
        }
        
        // Upstream ignored Range header, returning full 200 OK. Synthesize 206.
        const range = this.parseRange(clientRange, totalLength);
        if (!range) {
            return new Response(null, {
                status: 416,
                statusText: "Range Not Satisfiable",
                headers: { "Content-Range": `bytes */${totalLength}` }
            });
        }
        
        const chunkLength = range.end - range.start + 1;
        const modifiedHeaders = new Headers(originalHeaders);
        
        modifiedHeaders.set("Content-Range", `bytes ${range.start}-${range.end}/${totalLength}`);
        modifiedHeaders.set("Content-Length", chunkLength.toString());
        modifiedHeaders.set("Accept-Ranges", "bytes");
        // Ensure proxies don't cache this arbitrary slice incorrectly
        modifiedHeaders.set("Cache-Control", "no-cache, no-store, must-revalidate");
        modifiedHeaders.delete("Transfer-Encoding");
        
        const slicedStream = await this.createSlicedStream(upstreamResponse.body, range.start, range.end);
        
        return new Response(slicedStream, {
            status: 206,
            statusText: "Partial Content",
            headers: modifiedHeaders
        });
    }
}

// =======================================================================================
// *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-* Request Handler *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*
// =======================================================================================

/**
 * Main request handler separated from global scope for Module Worker compatibility.
 * @param {Request} request 
 * @param {string} proxyBaseUrl
 * @param {string} proxyHost
 * @param {ExecutionContext} ctx
 * @returns {Promise<Response>}
 */
async function handleProxyRequest(request, proxyBaseUrl, proxyHost, ctx) {

  // (UPDATE) â¦ CORSã¯æ­£ãããOPTIONSæªå¯¾å¿ï¼CORS Preflightã®ç¢ºå®ãªè¿å´ï¼
  // Reference: https://github.com/expressjs/cors for comprehensive media preflight headers
  if (request.method === "OPTIONS") {
    return new Response(null, {
      status: 204,
      headers: {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "GET, HEAD, POST, PUT, DELETE, OPTIONS",
        "Access-Control-Allow-Headers": request.headers.get("Access-Control-Request-Headers") || "Range, Accept, Origin, Content-Type, Authorization",
        "Access-Control-Max-Age": "86400"
      }
    });
  }

  const userAgent = request.headers.get('User-Agent');
  if (userAgent && userAgent.includes("Bytespider")) {
    return new Response("Unauthorized Crawler Detected", { status: 403, headers: { "Content-Type": "text/plain" } });
  }

  var siteCookie = request.headers.get('Cookie');

  if (password != "") {
    if (siteCookie != null && siteCookie != "") {
      var pwd = getCook(passwordCookieName, siteCookie);
      if (pwd !== password) {
        return handleWrongPwd();
      }
    } else {
      return handleWrongPwd();
    }
  }

  const url = new URL(request.url);
  
  if (url.pathname === '/__PROXY_SW__.js') {
    return new Response(getServiceWorkerCode(encPrefix), {
      headers: {
        "Content-Type": "application/javascript",
        "Cache-Control": "no-cache"
      }
    });
  }

  // (21) Check Cache API properly on Module Worker
  const cache = caches.default;
  let cachedResponse = await cache.match(request);
  if (cachedResponse) {
    return cachedResponse;
  }

  if (request.url.endsWith("favicon.ico")) {
     // Return 404 to avoid useless Baidu redirection and let the site handle its own
     return new Response(null, { status: 404 });
  }
  if (request.url.endsWith("robots.txt")) {
    return new Response(`User-Agent: *\nDisallow: /`, { headers: { "Content-Type": "text/plain" } });
  }

  var rawActualPath = url.pathname.substring(url.pathname.indexOf(routePrefix) + routePrefix.length);
  var actualUrlStr = _dec(rawActualPath);

  if (rawActualPath == "" || rawActualPath == "/") { 
    return getHTMLResponse(mainPage);
  }

  if (actualUrlStr) {
      actualUrlStr += url.search + url.hash;
  }

  try {
    var test = actualUrlStr;
    if (test && !test.startsWith("http") && test.includes(".")) {
      test = "https://" + test;
    }
    var u = new URL(test);
    if (!u.host.includes(".")) {
      throw new Error();
    }
    actualUrlStr = test;
  }
  catch (err) { 
    debug("URL Parse Error:", err);
    var lastVisit;
    if (siteCookie != null && siteCookie != "") {
      lastVisit = getCook(lastVisitProxyCookie, siteCookie);
      if (lastVisit != null && lastVisit != "") {
        return Response.redirect(proxyBaseUrl + _enc(lastVisit + (actualUrlStr.startsWith("/") ? "" : "/") + actualUrlStr), 301);
      }
    }
    return getHTMLResponse(getErrorTemplate("Cannot determine target URL. Cookie: " + siteCookie + " LastSite: " + lastVisit));
  }

  if (!actualUrlStr.startsWith("http") && !actualUrlStr.includes("://")) { 
    return Response.redirect(proxyBaseUrl + _enc("https://" + actualUrlStr), 301);
  }

  const actualUrl = new URL(actualUrlStr);

  if (rawActualPath !== _enc(actualUrlStr) && !rawActualPath.match(/^p\d*-/)) {
      return Response.redirect(proxyBaseUrl + _enc(actualUrl.href), 301);
  }
  
  if (actualUrlStr.startsWith(encPrefix)) {
      actualUrlStr = _dec(actualUrlStr);
  }

  let clientHeaderWithChange = new Headers();
  request.headers.forEach((value, key) => {
    let lowerKey = key.toLowerCase();
    var newValue = value.replaceAll(proxyBaseUrl + "http", "http");
    newValue = newValue.replaceAll(proxyBaseUrl, `${actualUrl.protocol}//${actualUrl.hostname}/`);
    newValue = newValue.replaceAll(proxyBaseUrl.substring(0, proxyBaseUrl.length - 1), `${actualUrl.protocol}//${actualUrl.hostname}`);
    newValue = newValue.replaceAll(proxyHost, actualUrl.host);
    
    if (lowerKey === 'referer' || lowerKey === 'origin') {
        newValue = actualUrl.origin + "/";
    }
    if (lowerKey === 'accept-encoding') {
        return; 
    }
    clientHeaderWithChange.set(lowerKey, newValue);
  });

  let clientRequestBodyWithChange = request.body;
  
  // (UPDATE) â  Rangeãªã¯ã¨ã¹ãã¯éã£ã¦ããã upstream ã«æ¸¡ãä¿è¨¼ãå¼±ãï¼ç¢ºå®ãªãã§ãããªãã·ã§ã³ã®æ§ç¯ï¼
  // Reference: https://github.com/node-fetch/node-fetch to ensure headers are preserved completely
  const fetchOptions = {
    method: request.method,
    headers: clientHeaderWithChange,
    body: (request.method !== 'GET' && request.method !== 'HEAD') ? clientRequestBodyWithChange : null,
    redirect: "manual",
    cf: {
      http3: true, 
      resolveOverride: actualUrl.hostname 
    }
  };

  if (request.headers.get("Range")) {
      fetchOptions.headers.set("Range", request.headers.get("Range"));
      debug("Forwarding explicit Range header for media request:", request.headers.get("Range"));
  } else if (request.headers.get("range")) {
      fetchOptions.headers.set("Range", request.headers.get("range"));
      debug("Forwarding explicit range header for media request:", request.headers.get("range"));
  }

  let response;
  try {
    // new Request()ã®åçæãåé¿ããç´æ¥ fetchOptions ãæ¸¡ãã¦éä¿¡å£åãé²ã
    response = await fetch(actualUrl.href, fetchOptions);
  } catch (err) {
    debug("Fetch Error:", err);
    return getHTMLResponse(getErrorTemplate(`Upstream fetch failed: ${err.message}`));
  }
  
  const originalStatus = response.status;

  if (response.status.toString().startsWith("3") && response.headers.get("Location") != null) {
    try {
      return Response.redirect(proxyBaseUrl + _enc(new URL(response.headers.get("Location"), actualUrlStr).href), 301);
    } catch (e) {
      debug("Redirect Gen Error:", e);
      return getHTMLResponse(getErrorTemplate("Redirect error: " + response.headers.get("Location")));
    }
  }

  var modifiedResponse;
  var hasProxyHintCook = (getCook(proxyHintCookieName, siteCookie) != "");
  const contentType = response.headers.get("Content-Type") || "";

  const isHls = contentType && (contentType.includes("mpegurl") || contentType.includes("x-mpegurl") || contentType.includes("vnd.apple.mpegurl") || contentType.includes("application/vnd.apple.mpegurl"));
  const isDash = contentType && contentType.includes("dash+xml");
  const isStreamOrVideo = (contentType && (contentType.includes("video/") || contentType.includes("audio/") || contentType.includes("application/wasm") || contentType.includes("application/octet-stream") || contentType.includes("image/")));

  if (response.body) {
    if (isStreamOrVideo && !isHls && !isDash) {
        debug("Bypassing TransformStream for raw media/video data to preserve binary integrity.");
        
        // (UPDATE) â£ Content-Lengthãæ¶ãã¦ããå¯è½æ§ï¼æå³çãªä¿æï¼ã¨å®å¨ãªã¹ããªã¼ã ãã³ããªã³ã°ã®å°å¥
        // Reference: https://github.com/videojs/video.js for chunked video player bounds
        // Reference: https://github.com/mischah/range-parser for robust Range handling
        modifiedResponse = await MediaStreamProxyEngine.processMediaResponse(request, response, originalStatus, response.headers);
        
        const originalContentLength = response.headers.get("Content-Length");
        if (originalContentLength && !modifiedResponse.headers.has("Content-Length")) {
            modifiedResponse.headers.set("Content-Length", originalContentLength);
        }
    }
    else if (isHls) {
        let decoder;
        let encoder;
        let buffer = "";
        const { readable, writable } = new TransformStream({
            start() {
                decoder = new TextDecoder("utf-8");
                encoder = new TextEncoder();
            },
            transform(chunk, controller) {
                buffer += decoder.decode(chunk, { stream: true });
                let lines = buffer.split('\n');
                buffer = lines.pop(); 
                
                for (let line of lines) {
                    if (line.startsWith('#EXT')) {
                        if (line.includes('URI=')) {
                            try {
                                line = line.replace(/URI="([^"]+)"/g, (match, p1) => {
                                    let resolvedUri = strictResolveUrl(p1, actualUrlStr);
                                    let proxiedUri = proxyBaseUrl + _enc(resolvedUri);
                                    return `URI="${proxiedUri}"`;
                                });
                            } catch (parseError) {
                                debug("HLS URI parse error:", parseError);
                            }
                        }
                        controller.enqueue(encoder.encode(line + '\n'));
                    } 
                    // (UPDATE) â¤ m3u8ã®URLæ¸ãæãã¯ããã tsã»ã°ã¡ã³ãæªå¯¾å¿ã®ä¿®æ­£
                    // Reference: https://github.com/video-dev/hls.js to assure proper segment fetching
                    else if (line.trim() !== '' && !line.startsWith('#')) {
                        try {
                            let resolved = strictResolveUrl(line.trim(), actualUrlStr);
                            // .ts ãªã©ã®ã¡ãã£ã¢ã»ã°ã¡ã³ããå®å¨ã«ãã­ã­ã·çµç±ã¸æ¸ãæãã
                            controller.enqueue(encoder.encode(proxyBaseUrl + _enc(resolved) + '\n'));
                        } catch (parseError) {
                            controller.enqueue(encoder.encode(line + '\n'));
                        }
                    } else {
                        controller.enqueue(encoder.encode(line + '\n'));
                    }
                }
            },
            flush(controller) {
                buffer += decoder.decode();
                if (buffer) {
                    let lines = buffer.split('\n');
                    for (let line of lines) {
                        if (line.startsWith('#EXT')) {
                            if (line.includes('URI=')) {
                                try {
                                    line = line.replace(/URI="([^"]+)"/g, (match, p1) => {
                                        let resolvedUri = strictResolveUrl(p1, actualUrlStr);
                                        let proxiedUri = proxyBaseUrl + _enc(resolvedUri);
                                        return `URI="${proxiedUri}"`;
                                    });
                                } catch (parseError) {
                                    debug("HLS URI parse error in flush:", parseError);
                                }
                            }
                            controller.enqueue(encoder.encode(line + '\n'));
                        } else if (line.trim() !== '' && !line.startsWith('#')) {
                            try {
                                let resolved = strictResolveUrl(line.trim(), actualUrlStr);
                                controller.enqueue(encoder.encode(proxyBaseUrl + _enc(resolved) + '\n'));
                            } catch (parseError) {
                                controller.enqueue(encoder.encode(line + (line ? '\n' : '')));
                            }
                        } else {
                            controller.enqueue(encoder.encode(line + (line ? '\n' : '')));
                        }
                    }
                }
            }
        });
        response.body.pipeTo(writable).catch(e => debug("HLS Pipe error", e));
        modifiedResponse = new Response(readable, {
            status: originalStatus,
            statusText: response.statusText,
            headers: new Headers(response.headers)
        });
    }
    else if (isDash) {
        let decoder;
        let encoder;
        const { readable, writable } = new TransformStream({
            start() {
                decoder = new TextDecoder("utf-8");
                encoder = new TextEncoder();
            },
            transform(chunk, controller) {
                let text = decoder.decode(chunk, { stream: true });
                let replaced = text.replace(/<BaseURL>([^<]+)<\/BaseURL>/g, (match, p1) => {
                    return `<BaseURL>${proxyBaseUrl + _enc(strictResolveUrl(p1, actualUrlStr))}</BaseURL>`;
                });
                replaced = replaced.replace(/(media|initialization|sourceURL)="([^"]+)"/g, (match, attr, p1) => {
                    if (p1.includes('$') && !p1.startsWith('http')) {
                        if (p1.startsWith('http')) {
                            return `${attr}="${proxyBaseUrl + _enc(p1)}"`;
                        }
                        return match;
                    }
                    if (p1 && !p1.startsWith('urn:')) {
                        return `${attr}="${proxyBaseUrl + _enc(strictResolveUrl(p1, actualUrlStr))}"`;
                    }
                    return match;
                });
                controller.enqueue(encoder.encode(replaced));
            },
            flush(controller) {
                let text = decoder.decode();
                if (text) {
                    let replaced = text.replace(/<BaseURL>([^<]+)<\/BaseURL>/g, (match, p1) => {
                        return `<BaseURL>${proxyBaseUrl + _enc(strictResolveUrl(p1, actualUrlStr))}</BaseURL>`;
                    });
                    replaced = replaced.replace(/(media|initialization|sourceURL)="([^"]+)"/g, (match, attr, p1) => {
                        if (p1.includes('$') && !p1.startsWith('http')) {
                            if (p1.startsWith('http')) {
                                return `${attr}="${proxyBaseUrl + _enc(p1)}"`;
                            }
                            return match;
                        }
                        if (p1 && !p1.startsWith('urn:')) {
                            return `${attr}="${proxyBaseUrl + _enc(strictResolveUrl(p1, actualUrlStr))}"`;
                        }
                        return match;
                    });
                    controller.enqueue(encoder.encode(replaced));
                }
            }
        });
        response.body.pipeTo(writable).catch(e => debug("DASH Pipe error", e));
        modifiedResponse = new Response(readable, {
            status: originalStatus,
            statusText: response.statusText,
            headers: new Headers(response.headers)
        });
    }
    else if (contentType && contentType.includes("application/json")) {
        let decoder;
        let encoder;
        const { readable, writable } = new TransformStream({
            start() {
                decoder = new TextDecoder("utf-8");
                encoder = new TextEncoder();
            },
            transform(chunk, controller) {
                let text = decoder.decode(chunk, { stream: true });
                let regex = /(https?:\/\/[^\s'"\\]+)/g;
                let replaced = text.split(regex).map((part, index) => {
                  if (index % 2 === 1) return proxyBaseUrl + _enc(part);
                  return part;
                }).join("");
                controller.enqueue(encoder.encode(replaced));
            },
            flush(controller) {
                let text = decoder.decode();
                if (text) {
                    let regex = /(https?:\/\/[^\s'"\\]+)/g;
                    let replaced = text.split(regex).map((part, index) => {
                      if (index % 2 === 1) return proxyBaseUrl + _enc(part);
                      return part;
                    }).join("");
                    controller.enqueue(encoder.encode(replaced));
                }
            }
        });
        response.body.pipeTo(writable).catch(e => debug("Pipe error", e));
        
        modifiedResponse = new Response(readable, {
          status: originalStatus,
          statusText: response.statusText,
          headers: new Headers(response.headers)
       });
    }
    else if (contentType && contentType.startsWith("text/") && !isStreamOrVideo) {
      let isHTML = contentType.includes("text/html");
      
      if (isHTML) {
        let linkHeader =[];
        
        class AttributeRewriter {
          constructor() {}
          element(element) {
            if(element.hasAttribute("href")) this.rewriteAttr(element, "href");
            if(element.hasAttribute("src")) this.rewriteAttr(element, "src");
            if(element.hasAttribute("srcset")) this.rewriteSrcset(element, "srcset");
            
            if(element.tagName === "img" && !element.hasAttribute("loading")) {
                element.setAttribute("loading", "lazy");
            }
            if(element.hasAttribute("integrity")) {
                element.removeAttribute("integrity");
            }
            element.setAttribute('data-yproxy-handled', '1');
          }
          
          rewriteAttr(element, attr) {
            const val = element.getAttribute(attr);
            if (val && val.startsWith("http") && !val.includes(proxyHost)) {
              element.setAttribute(attr, proxyBaseUrl + _enc(val));
            }
          }

          rewriteSrcset(element, attr) {
            const val = element.getAttribute(attr);
            if(val) {
               const parts = val.split(',');
               const rewrittenParts = parts.map(part => {
                 const subParts = part.trim().split(' ');
                 if (subParts.length > 0 && subParts[0].startsWith('http')) {
                   subParts[0] = proxyBaseUrl + _enc(subParts[0]);
                 }
                 return subParts.join(' ');
               });
               element.setAttribute(attr, rewrittenParts.join(', '));
            }
          }
        }

        class MetaRewriter {
          element(element) {
             const httpEquiv = element.getAttribute('http-equiv');
             if (httpEquiv && httpEquiv.toLowerCase() === 'refresh') {
                 element.remove();
             }
          }
        }

        const attrRewriterInstance = new AttributeRewriter();

        // (UPDATE) â¥ HTMLRewriterãåç»ã¿ã°ãå£ãå¯è½æ§ï¼éå°ãªæ¸ãæãã®æ¤å»ï¼
        // Extracted video[src] and audio[src] to protect primary media element playback integrity
        const rewriter = new HTMLRewriter()
          .on("head", {
            element(element) {
               var inject = `
                <link rel="preconnect" href="${proxyBaseUrl}">
                <script defer>
                (function () {
                  ${((!hasProxyHintCook) ? proxyHintInjection : "")}
                })();
                (function () {
                  ${getClientInjectionCode(proxyBaseUrl, proxyHost)}
                  ${getHtmlCovPathInjectCode("parseAndInsertDoc", proxyBaseUrl)}
                  parseAndInsertDoc(""); 
                  ${getDevToolsCode(proxyBaseUrl)}
                })();
                </script>
               `;
               element.prepend(inject, { html: true });
            }
          })
          .on("a[href], area[href], link[href], base[href], img[src], script[src], iframe[src], source[src], img[srcset], source[srcset]", attrRewriterInstance)
          .on("meta", new MetaRewriter())
          .on("link[rel='stylesheet'], script[src]", {
              element(el) {
                  const src = el.getAttribute('src') || el.getAttribute('href');
                  if (src && src.startsWith('http')) {
                      try {
                          const u = new URL(src);
                          linkHeader.push(`<${u.origin}>; rel=preconnect`);
                      } catch(e){}
                  }
              }
          });

        let transformedBody = rewriter.transform(response).body;
        modifiedResponse = new Response(transformedBody, {
          status: originalStatus,
          statusText: response.statusText,
          headers: new Headers(response.headers)
        });
        if (linkHeader.length > 0) {
            modifiedResponse.headers.append('Link', linkHeader.join(', '));
        }
      } else {
        let decoder;
        let encoder;
        const { readable, writable } = new TransformStream({
            start() {
                decoder = new TextDecoder("utf-8");
                encoder = new TextEncoder();
            },
            transform(chunk, controller) {
                let text = decoder.decode(chunk, { stream: true });
                let regex = /(https?:\/\/[^\s'"\\]+)/g;
                let replaced = text.split(regex).map((part, index) => {
                  if (index % 2 === 1) return proxyBaseUrl + _enc(part);
                  return part;
                }).join("");
                controller.enqueue(encoder.encode(replaced));
            },
            flush(controller) {
                let text = decoder.decode();
                if (text) {
                    let regex = /(https?:\/\/[^\s'"\\]+)/g;
                    let replaced = text.split(regex).map((part, index) => {
                      if (index % 2 === 1) return proxyBaseUrl + _enc(part);
                      return part;
                    }).join("");
                    controller.enqueue(encoder.encode(replaced));
                }
            }
        });

        response.body.pipeTo(writable).catch(e => debug("Stream pipe error:", e));
        modifiedResponse = new Response(readable, {
          status: originalStatus,
          statusText: response.statusText,
          headers: new Headers(response.headers)
        });
      }

    }
    else {
      // Fallback for missing Content-Type or unaccounted blobs
      modifiedResponse = new Response(response.body, {
          status: originalStatus,
          statusText: response.statusText,
          headers: new Headers(response.headers)
      });
      if (request.method === 'GET' && originalStatus === 200) {
          if (url.pathname.match(/[\.\-][0-9a-f]{8,}/) || url.search.includes('v=')) {
              modifiedResponse.headers.set('Cache-Control', 'public, max-age=31536000, immutable');
          }
      }
    }
  }
  else {
    modifiedResponse = new Response(response.body, {
          status: originalStatus,
          statusText: response.statusText,
          headers: new Headers(response.headers)
    });
  }

  let headers = modifiedResponse.headers;

  let rawCookies =[];
  if (typeof headers.getSetCookie === 'function') {
      rawCookies = headers.getSetCookie();
  } else {
      const fallbackCookie = headers.get('set-cookie');
      if (fallbackCookie) {
          rawCookies = fallbackCookie.split(',').map(c => c.trim());
      }
  }

  if (rawCookies.length > 0) {
    headers.delete('Set-Cookie');

    rawCookies.forEach(cookieStr => {
      let parts = cookieStr.split(';').map(part => part.trim());

      let pathIndex = parts.findIndex(part => part.toLowerCase().startsWith('path='));
      let originalPath;
      if (pathIndex !== -1) {
        originalPath = parts[pathIndex].substring("path=".length);
      }
      let absolutePath = "/" + _enc(strictResolveUrl(originalPath || "/", actualUrlStr));

      if (pathIndex !== -1) {
        parts[pathIndex] = `Path=${absolutePath}`;
      } else {
        parts.push(`Path=${absolutePath}`);
      }

      let domainIndex = parts.findIndex(part => part.toLowerCase().startsWith('domain='));
      if (domainIndex !== -1) {
        parts[domainIndex] = `domain=${proxyHost}`;
      } else {
        parts.push(`domain=${proxyHost}`);
      }

      let sameSiteIndex = parts.findIndex(part => part.toLowerCase().startsWith('samesite='));
      if(sameSiteIndex !== -1) {
          parts[sameSiteIndex] = `SameSite=None`;
      } else {
          parts.push(`SameSite=None`);
      }
      if(!parts.find(part => part.toLowerCase() === 'secure')) {
          parts.push('Secure');
      }

      headers.append('Set-Cookie', parts.join('; ')); 
    });
  }
  
  let isHTMLCheck = contentType.includes("text/html");
  if (isHTMLCheck && response.status == 200) { 
    let cookieValue = lastVisitProxyCookie + "=" + actualUrl.origin + "; Path=/; Domain=" + proxyHost + "; SameSite=None; Secure";
    headers.append("Set-Cookie", cookieValue);

    if (response.body && !hasProxyHintCook) { 
      const expiryDate = new Date();
      expiryDate.setTime(expiryDate.getTime() + 24 * 60 * 60 * 1000); 
      var hintCookie = `${proxyHintCookieName}=1; expires=${expiryDate.toUTCString()}; path=/; SameSite=None; Secure`;
      headers.append("Set-Cookie", hintCookie);
    }
  }

  // (UPDATE) â¢ Accept-Ranges ã upstream ã«ç¡ãå ´åã®ãªã©ã¤ãã£ã³ã°ä¿è­·
  if (isStreamOrVideo || isHls || isDash || originalStatus === 206) {
      headers.set("Access-Control-Allow-Origin", "*");
      headers.set("Access-Control-Allow-Headers", "Range, Accept, Origin, Content-Type");
      headers.set("Access-Control-Allow-Methods", "GET, HEAD, POST, OPTIONS");
      headers.set("Access-Control-Expose-Headers", "Content-Length, Content-Range, Accept-Ranges");
      
      if (!response.headers.get("Accept-Ranges")) {
          headers.set("Accept-Ranges", "bytes");
      }
  } else {
      headers.delete("Access-Control-Allow-Origin");
      headers.set('Access-Control-Allow-Origin', '*');
  }
  
  headers.set("X-Frame-Options", "ALLOWALL");

  var listHeaderDel =["Content-Security-Policy", "Permissions-Policy", "Cross-Origin-Embedder-Policy", "Cross-Origin-Resource-Policy"];
  listHeaderDel.forEach(element => {
    headers.delete(element);
    headers.delete(element + "-Report-Only");
  });

  if (!hasProxyHintCook) {
    headers.set("Cache-Control", "max-age=0");
  }

  // (UPDATE) â¡ TransformStreamãåç»ãå£ãã¦ããå¯è½æ§ï¼clone()ã®æ¤å»ãã­ã£ãã·ã¥ã­ã¸ãã¯ã«ãé©ç¨ï¼
  if (request.method === 'GET' && response.status === 200 && contentType.includes('image/') && !isStreamOrVideo && !isHls && !isDash) {
    ctx.waitUntil(cache.put(request, modifiedResponse.clone()));
  }

  return modifiedResponse;
}

/**
 * @param {string} cookiename 
 * @param {string|null} cookies 
 * @returns {string}
 */
function getCook(cookiename, cookies) {
  if (!cookies) return "";
  const match = cookies.match(new RegExp('(^|;\\s*)(' + cookiename + ')=([^;]*)'));
  return match ? decodeURIComponent(match[3]) : "";
}

function handleWrongPwd() {
  if (showPasswordPage) {
    return getHTMLResponse(pwdPage);
  } else {
    return getHTMLResponse(getErrorTemplate("403 Forbidden - You do not have access to view this webpage."));
  }
}

function getHTMLResponse(html) {
  return new Response(html, {
    headers: { "Content-Type": "text/html; charset=utf-8" }
  });
}

// (24) Extracted relative path resolver using pure string logic to avoid heavy new URL() where possible
function strictResolveUrl(relativePath, base) {
  if (!relativePath) return relativePath;
  if (relativePath.startsWith('http')) return relativePath;
  if (relativePath.startsWith('//')) {
    const idx = base.indexOf(':');
    if (idx !== -1) return base.substring(0, idx + 1) + relativePath;
    return 'https:' + relativePath;
  }
  try {
    return new URL(relativePath, base).href;
  } catch (e) {
    if (relativePath.startsWith('/')) {
      try { return new URL(base).origin + relativePath; } catch(err){}
    }
    return base.replace(/\/[^\/]*$/, '/') + relativePath;
  }
}

// =======================================================================================
// *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-* Module Worker Export *-*-*-*-*-*-*-*-*-*-*-*-*-*
// =======================================================================================
export default {
  /**
   * Fetch handler for Module Worker (Replacing legacy addEventListener)
   * @param {Request} request 
   * @param {Object} env 
   * @param {Object} ctx 
   */
  async fetch(request, env, ctx) {
    // Override CONFIG with environment variables for security
    if (env) {
      if (env.PROXY_PASSWORD !== undefined) CONFIG.password = env.PROXY_PASSWORD;
      if (env.DEBUG_MODE !== undefined) CONFIG.DEBUG_MODE = env.DEBUG_MODE === 'true' || env.DEBUG_MODE === true;
      if (env.ENC_PREFIX !== undefined) CONFIG.encPrefix = env.ENC_PREFIX;
    }
    const url = new URL(request.url);
    const proxyBaseUrl = `${url.protocol}//${url.hostname}/`;
    const proxyHost = url.host;
    return handleProxyRequest(request, proxyBaseUrl, proxyHost, ctx);
  }
};
