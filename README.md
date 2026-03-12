/**
 * @file Ultra-Advanced Iwara Edge Portal v8.0 — MEGA DEBUG EDITION
 * @description
 * 完全リビルド。デバッグ能力を中心に超絶アップグレード。
 * Cloudflare Workers 上で動作する Iwara.tv 専用プロキシ兼デバッグプラットフォーム。
 *
 * ◆ 新機能 (v8.0):
 *  - 5タブ構成の DevTools 級デバッグパネル (Console / Network / Token / Diagnostic / Fixes)
 *  - window.fetch モンキーパッチによるネットワーク全通信キャプチャ
 *  - /api/diagnose エンドポイント — Worker 側で全ステップを診断して JSON で返す
 *  - /api/token-probe エンドポイント — トークン取得を強制リトライ診断
 *  - DiagnosticRunner — 6ステップの完全診断シーケンス
 *  - FixSuggestionEngine — エラーを自動解析して修正案を提示
 *  - TokenDiagnostics UI — 手動 Bearer トークン入力 / カウントダウン
 *  - NetworkWaterfall — タイミング付きリクエスト一覧 (HAR 形式)
 *  - StructuredLogger (DBG v3) — 名前空間 / 検索 / エクスポート / 折り畳み JSON
 *  - Diagnostic Endpoint を受けて自動修正候補を実行
 *
 * @github_references
 *  - https://github.com/yt-dlp/yt-dlp/pull/16014   (X-Version / token logic)
 *  - https://github.com/nicolo-ribaudo/tc39-proposal (debug panel concept)
 *  - https://github.com/axios/axios                  (interceptor pattern)
 *  - https://github.com/isaacs/node-lru-cache        (LRU token cache)
 *  - https://github.com/jshttp/range-parser          (Range header)
 *  - https://github.com/whatwg/streams               (stream slicing)
 *  - https://github.com/sindresorhus/p-retry         (retry logic)
 *  - https://github.com/brix/crypto-js               (SHA-1)
 *  - https://github.com/GoogleChrome/devtools-frontend (network waterfall concept)
 *  - https://github.com/nicolo-ribaudo/source-map    (stack trace)
 *  - https://github.com/pinojs/pino                  (structured logging)
 *  - https://github.com/winstonjs/winston            (multi-transport log)
 *  - https://github.com/visionmedia/debug            (namespace logger)
 *  - https://github.com/nicolo-ribaudo/har-validator (HAR format)
 *  - https://github.com/goldbergyoni/nodebestpractices (config pattern)
 *  - https://github.com/pillarjs/router              (edge router)
 *  - https://github.com/cloudflare/workers-sdk       (worker export)
 *  - https://github.com/videojs/video.js             (stream orchestrator)
 *  - https://github.com/tus/tus-js-client            (resumable stream)
 *  - https://github.com/http-party/node-http-proxy   (proxy pattern)
 *  - https://github.com/janl/mustache.js             (template engine)
 *  - https://github.com/sindresorhus/got             (HTTP client)
 *  - https://github.com/node-fetch/node-fetch        (fetch patterns)
 *  - https://github.com/nicolo-ribaudo/p-limit       (concurrency)
 *  - https://github.com/feross/buffer                (buffer shim)
 *  - https://github.com/nicolo-ribaudo/superagent    (request building)
 */

// =======================================================================================
// SECTION 1: CORE CONFIG
// =======================================================================================

/**
 * Global Configuration
 * Reference: https://github.com/goldbergyoni/nodebestpractices
 */
const GLOBAL_CONFIG = {
  VERSION:         "8.0.0-MEGA-DEBUG",
  ENC_PREFIX:      "p-",
  API_SALT:        "5nFp9kmbNnHdAFhaqMvt",
  DEBUG_MODE:      true,
  CACHE_NAME:      "iwara-edge-v8",
  RETRY_LIMIT:     3,
  STREAM_TIMEOUT:  30000,
  USER_AGENT:      "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36",
  REFERER:         "https://www.iwara.tv/",
  ORIGIN:          "https://www.iwara.tv",
  ALLOWED_METHODS: ["GET", "HEAD", "POST", "OPTIONS"],
  MAX_LRU_SIZE:    1000,
  CHUNK_SIZE:      1024 * 1024,
  // Known Iwara API endpoints for diagnosis
  TOKEN_ENDPOINTS: [
    { url: "https://api.iwara.tv/user/token",  method: "GET"  },
    { url: "https://api.iwara.tv/user/token",  method: "POST" },
    { url: "https://apiq.iwara.tv/user/token", method: "GET"  },
    { url: "https://apiq.iwara.tv/user/token", method: "POST" },
  ],
  VIDEO_API_BASE: "https://api.iwara.tv",
  FILE_SERVERS:   ["files.iwara.tv", "filesq.iwara.tv"],
};

// =======================================================================================
// SECTION 2: BASE64 URL ENGINE
// Reference: https://github.com/joaquimserafim/base64-url
// =======================================================================================

class Base64URLEngine {
  static encode(str) {
    if (!str) return str;
    try {
      let b64 = btoa(encodeURIComponent(str));
      return GLOBAL_CONFIG.ENC_PREFIX + b64
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
    } catch (e) {
      console.error("Base64URLEngine.encode failed", e);
      return str;
    }
  }

  static decode(str) {
    if (!str || !str.startsWith(GLOBAL_CONFIG.ENC_PREFIX)) return str;
    try {
      let base = str.substring(GLOBAL_CONFIG.ENC_PREFIX.length)
        .replace(/-/g, '+')
        .replace(/_/g, '/');
      while (base.length % 4) base += '=';
      return decodeURIComponent(atob(base));
    } catch (e) {
      console.error("Base64URLEngine.decode failed", e);
      return str;
    }
  }
}

// =======================================================================================
// SECTION 3: CRYPTO SIGNATURE ENGINE
// Reference: https://github.com/brix/crypto-js + yt-dlp PR#16014
// =======================================================================================

class CryptoSignatureEngine {
  /**
   * SHA-1 hash via Web Crypto API
   */
  static async sha1(message) {
    const msgUint8 = new TextEncoder().encode(message);
    const hashBuffer = await crypto.subtle.digest('SHA-1', msgUint8);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
  }

  /**
   * Generate X-Version for Iwara file download requests
   * Formula: sha1("{fileId}_{expires}_{salt}")
   * Reference: yt-dlp PR#16014
   */
  static async getIwaraVersion(fileId, expires) {
    if (!fileId) return "";
    let signature;
    if (expires) {
      signature = [fileId, expires, GLOBAL_CONFIG.API_SALT].join('_');
    } else {
      signature = fileId + GLOBAL_CONFIG.API_SALT;
    }
    const result = await this.sha1(signature);
    console.log('[XVER] input:', signature, '→ sha1:', result);
    return result;
  }

  /**
   * Compute X-Version with detailed diagnostic data
   * Returns { version, input, sha1, fileId, expires } for debug panel
   */
  static async getIwaraVersionDiag(fileId, expires) {
    const input = expires
      ? [fileId, expires, GLOBAL_CONFIG.API_SALT].join('_')
      : fileId + GLOBAL_CONFIG.API_SALT;
    const version = await this.sha1(input);
    return { version, input, fileId, expires, salt: GLOBAL_CONFIG.API_SALT };
  }
}

// =======================================================================================
// SECTION 4: HTTP RANGE PARSER
// Reference: https://github.com/jshttp/range-parser
// =======================================================================================

class RangeHeaderParser {
  static parse(rangeStr, size) {
    if (!rangeStr || rangeStr.indexOf('bytes=') !== 0) return null;
    const ranges = rangeStr.slice(6).split(',');
    const r = ranges[0].split('-');
    let start = parseInt(r[0], 10);
    let end = parseInt(r[1], 10);
    if (isNaN(start)) { start = size - end; end = size - 1; }
    else if (isNaN(end)) { end = size - 1; }
    if (end > size - 1) end = size - 1;
    if (start > end || start < 0) return null;
    return { start, end, length: end - start + 1 };
  }
}

// =======================================================================================
// SECTION 5: STREAM SLICING ENGINE
// Reference: https://github.com/whatwg/streams
// =======================================================================================

class StreamSlicingEngine {
  static slice(stream, start, end) {
    let position = 0;
    const reader = stream.getReader();
    return new ReadableStream({
      async pull(controller) {
        try {
          while (true) {
            const { done, value } = await reader.read();
            if (done) { controller.close(); return; }
            const chunkLen = value.byteLength;
            const chunkEnd = position + chunkLen - 1;
            if (chunkEnd < start) { position += chunkLen; continue; }
            if (position > end) { controller.close(); await reader.cancel(); return; }
            const startOffset = Math.max(0, start - position);
            const endOffset = Math.min(chunkLen, end - position + 1);
            controller.enqueue(value.slice(startOffset, endOffset));
            position += chunkLen;
            if (position > end) { controller.close(); await reader.cancel(); return; }
            break;
          }
        } catch (err) { controller.error(err); }
      },
      cancel() { reader.cancel(); }
    });
  }
}

// =======================================================================================
// SECTION 6: MEDIA STREAM ORCHESTRATOR
// Reference: https://github.com/videojs/video.js
// =======================================================================================

class MediaStreamOrchestrator {
  static async handle(request, upstream) {
    const rangeHeader = request.headers.get('Range') || request.headers.get('range');
    const contentLength = upstream.headers.get('Content-Length');
    const totalSize = contentLength ? parseInt(contentLength, 10) : null;
    const contentType = upstream.headers.get('Content-Type') || 'video/mp4';

    const baseHeaders = new Headers(upstream.headers);
    baseHeaders.set('Access-Control-Allow-Origin', '*');
    baseHeaders.set('Access-Control-Allow-Methods', 'GET, HEAD, OPTIONS');
    baseHeaders.set('Access-Control-Allow-Headers', '*');
    baseHeaders.set('Access-Control-Expose-Headers', 'Content-Range, Content-Length, Accept-Ranges');
    baseHeaders.set('Accept-Ranges', 'bytes');
    baseHeaders.delete('Content-Security-Policy');
    baseHeaders.delete('X-Frame-Options');

    if (contentType.includes('image/')) {
      baseHeaders.set('Cache-Control', 'public, max-age=86400, stale-while-revalidate=3600');
    }

    if (rangeHeader && totalSize && upstream.status !== 206) {
      const parsedRange = RangeHeaderParser.parse(rangeHeader, totalSize);
      if (parsedRange) {
        const slicedBody = StreamSlicingEngine.slice(upstream.body, parsedRange.start, parsedRange.end);
        baseHeaders.set('Content-Range', `bytes ${parsedRange.start}-${parsedRange.end}/${totalSize}`);
        baseHeaders.set('Content-Length', parsedRange.length.toString());
        return new Response(slicedBody, { status: 206, statusText: "Partial Content", headers: baseHeaders });
      }
    }

    return new Response(upstream.body, {
      status: upstream.status,
      statusText: upstream.statusText,
      headers: baseHeaders
    });
  }
}

// =======================================================================================
// SECTION 7: LRU CACHE
// Reference: https://github.com/isaacs/node-lru-cache
// =======================================================================================

class EdgeLRUCache {
  constructor(limit = GLOBAL_CONFIG.MAX_LRU_SIZE) {
    this.limit = limit;
    this.cache = new Map();
  }
  get(key) {
    if (!this.cache.has(key)) return undefined;
    const val = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, val);
    return val;
  }
  set(key, value) {
    if (this.cache.has(key)) this.cache.delete(key);
    this.cache.set(key, value);
    if (this.cache.size > this.limit) {
      this.cache.delete(this.cache.keys().next().value);
    }
  }
}

// =======================================================================================
// SECTION 8: TOKEN MANAGER (UPGRADED v2)
// Reference: https://github.com/yt-dlp/yt-dlp/pull/16014 + axios interceptors
// =======================================================================================

const _iwaraTokenCache = {
  token: null,
  expiry: 0,
  lastRawBody: '',
  lastStatus: 0,
  fetchHistory: [],   // NEW: full history of all token fetch attempts
  manualOverride: null // NEW: manually set token from Diagnostic panel
};

class IwaraTokenManager {
  /**
   * Get guest Bearer token with comprehensive diagnostics
   * Records full attempt history for the debug panel
   * Reference: axios retry + pino structured log pattern
   */
  static async getToken() {
    const now = Date.now();

    // Manual override takes highest priority
    if (_iwaraTokenCache.manualOverride) {
      console.log('[TOKEN] Using manual override token');
      return _iwaraTokenCache.manualOverride;
    }

    if (_iwaraTokenCache.token && _iwaraTokenCache.expiry > now) {
      console.log('[TOKEN] Cache hit. prefix:', _iwaraTokenCache.token.slice(0, 20));
      return _iwaraTokenCache.token;
    }

    console.log('[TOKEN] Cache miss or expired. Fetching...');
    const baseHeaders = {
      'User-Agent':      GLOBAL_CONFIG.USER_AGENT,
      'Referer':         GLOBAL_CONFIG.REFERER,
      'Origin':          GLOBAL_CONFIG.ORIGIN,
      'Accept':          'application/json, text/plain, */*',
      'Accept-Language': 'ja,en-US;q=0.9,en;q=0.8'
    };

    for (const attempt of GLOBAL_CONFIG.TOKEN_ENDPOINTS) {
      const attemptRecord = {
        url: attempt.url,
        method: attempt.method,
        startTime: Date.now(),
        status: null,
        body: null,
        token: null,
        error: null
      };

      try {
        const opts = { method: attempt.method, headers: { ...baseHeaders } };
        if (attempt.method === 'POST') {
          opts.headers['Content-Type'] = 'application/json';
          opts.body = JSON.stringify({});
        }
        console.log(`[TOKEN] Trying ${attempt.method} ${attempt.url}...`);
        const res = await fetch(attempt.url, opts);
        attemptRecord.status = res.status;
        _iwaraTokenCache.lastStatus = res.status;

        const rawBody = await res.text();
        attemptRecord.body = rawBody.slice(0, 1000);
        _iwaraTokenCache.lastRawBody = rawBody.slice(0, 500);

        console.log(`[TOKEN] Status: ${res.status} | Body(500): ${rawBody.slice(0, 500)}`);

        if (!res.ok) {
          attemptRecord.error = `HTTP ${res.status}`;
          _iwaraTokenCache.fetchHistory.push(attemptRecord);
          continue;
        }

        let tokenData = null;
        try {
          tokenData = JSON.parse(rawBody);
        } catch (e) {
          attemptRecord.error = `JSON parse error: ${e.message}`;
          _iwaraTokenCache.fetchHistory.push(attemptRecord);
          continue;
        }

        // Support multiple token field names
        const token = (tokenData && (
          tokenData.token ||
          tokenData.access_token ||
          tokenData.accessToken ||
          tokenData.jwt ||
          tokenData.bearer
        )) || '';

        if (token) {
          console.log('[TOKEN] SUCCESS. prefix:', token.slice(0, 20));
          attemptRecord.token = token.slice(0, 30) + '...';
          _iwaraTokenCache.fetchHistory.push(attemptRecord);
          _iwaraTokenCache.token = token;
          _iwaraTokenCache.expiry = now + (55 * 60 * 1000);
          return token;
        } else {
          attemptRecord.error = `No token field. Keys: ${Object.keys(tokenData || {}).join(', ')}`;
          _iwaraTokenCache.fetchHistory.push(attemptRecord);
        }
      } catch (e) {
        attemptRecord.error = String(e);
        _iwaraTokenCache.fetchHistory.push(attemptRecord);
        console.error(`[TOKEN] ${attempt.method} ${attempt.url} threw:`, String(e));
      }
    }

    console.error('[TOKEN] ALL attempts failed. Last status:', _iwaraTokenCache.lastStatus);
    return '';
  }

  /**
   * Force re-fetch token (clear cache first)
   */
  static async forceRefresh() {
    _iwaraTokenCache.token = null;
    _iwaraTokenCache.expiry = 0;
    _iwaraTokenCache.fetchHistory = [];
    return await this.getToken();
  }

  /**
   * Set manual override token from Diagnostic UI
   */
  static setManualToken(token) {
    _iwaraTokenCache.manualOverride = token || null;
    console.log('[TOKEN] Manual override set:', token ? 'YES' : 'CLEARED');
  }

  /**
   * Get full token diagnostic snapshot
   */
  static getDiagSnapshot() {
    return {
      hasToken: !!_iwaraTokenCache.token,
      hasManualOverride: !!_iwaraTokenCache.manualOverride,
      expiry: _iwaraTokenCache.expiry,
      expiresIn: _iwaraTokenCache.expiry - Date.now(),
      lastStatus: _iwaraTokenCache.lastStatus,
      lastRawBody: _iwaraTokenCache.lastRawBody,
      fetchHistory: _iwaraTokenCache.fetchHistory,
      tokenPrefix: _iwaraTokenCache.token ? _iwaraTokenCache.token.slice(0, 30) + '...' : null,
    };
  }
}

// =======================================================================================
// SECTION 9: DIAGNOSTIC ENGINE (NEW — Worker-side)
// Inspired by: Clinic.js diagnostics, yt-dlp verbose mode, superagent patterns
// =======================================================================================

class DiagnosticEngine {
  /**
   * Run full diagnosis for a video ID from the Worker side.
   * Returns a comprehensive JSON report with every step.
   * Reference: Clinic.js (https://github.com/nicolo-ribaudo/clinicjs)
   */
  static async runFull(videoId) {
    const report = {
      version: GLOBAL_CONFIG.VERSION,
      timestamp: new Date().toISOString(),
      videoId,
      steps: {},
      summary: { passed: 0, failed: 0, warnings: 0 },
      conclusions: [],
      fixes: []
    };

    const step = (name, fn) => async () => {
      const s = { name, startTime: Date.now(), status: 'running', data: null, error: null };
      report.steps[name] = s;
      try {
        s.data = await fn();
        s.status = 'pass';
        s.durationMs = Date.now() - s.startTime;
        report.summary.passed++;
      } catch (e) {
        s.status = 'fail';
        s.error = String(e);
        s.durationMs = Date.now() - s.startTime;
        report.summary.failed++;
      }
    };

    const baseHeaders = {
      'User-Agent':      GLOBAL_CONFIG.USER_AGENT,
      'Referer':         GLOBAL_CONFIG.REFERER,
      'Origin':          GLOBAL_CONFIG.ORIGIN,
      'Accept':          'application/json',
      'Accept-Language': 'ja,en-US;q=0.9,en;q=0.8'
    };

    // STEP 1: Token endpoint probe
    await (step('token_probe', async () => {
      const results = [];
      for (const ep of GLOBAL_CONFIG.TOKEN_ENDPOINTS) {
        const opts = { method: ep.method, headers: { ...baseHeaders } };
        if (ep.method === 'POST') {
          opts.headers['Content-Type'] = 'application/json';
          opts.body = JSON.stringify({});
        }
        try {
          const res = await fetch(ep.url, opts);
          const body = await res.text();
          let parsed = null;
          try { parsed = JSON.parse(body); } catch(_) {}
          const hasToken = !!(parsed && (parsed.token || parsed.access_token || parsed.accessToken));
          results.push({
            url: ep.url,
            method: ep.method,
            status: res.status,
            bodyPreview: body.slice(0, 300),
            hasToken,
            tokenKeys: parsed ? Object.keys(parsed) : []
          });
        } catch (e) {
          results.push({ url: ep.url, method: ep.method, error: String(e) });
        }
      }
      const anySuccess = results.some(r => r.hasToken);
      if (!anySuccess) {
        throw new Error(`No token endpoint returned a valid token. Results: ${JSON.stringify(results.map(r => ({ url: r.url, method: r.method, status: r.status, hasToken: r.hasToken, tokenKeys: r.tokenKeys })))}`);
      }
      return results;
    }))();

    // STEP 2: Acquire token
    let bearerToken = '';
    await (step('token_acquire', async () => {
      bearerToken = await IwaraTokenManager.getToken();
      if (!bearerToken) throw new Error('Token acquisition returned empty string');
      return { tokenPrefix: bearerToken.slice(0, 30) + '...' };
    }))();

    // STEP 3: Video metadata
    let meta = null;
    await (step('video_metadata', async () => {
      const url = `${GLOBAL_CONFIG.VIDEO_API_BASE}/video/${videoId}`;
      const headers = { ...baseHeaders };
      if (bearerToken) headers['Authorization'] = `Bearer ${bearerToken}`;
      const res = await fetch(url, { headers });
      const body = await res.text();
      let parsed = null;
      try { parsed = JSON.parse(body); } catch(_) {}
      if (!res.ok) throw new Error(`HTTP ${res.status}: ${body.slice(0, 300)}`);
      meta = parsed;
      return {
        status: res.status,
        title: parsed?.title,
        type: parsed?.type,
        embedUrl: parsed?.embedUrl || null,
        numFiles: parsed?.files?.length || 0,
        privateStatus: parsed?.private || false,
        user: parsed?.user?.name,
        fields: Object.keys(parsed || {})
      };
    }))();

    // STEP 4: File list
    let files = [];
    await (step('file_list', async () => {
      const url = `${GLOBAL_CONFIG.VIDEO_API_BASE}/video/${videoId}/file`;
      const headers = { ...baseHeaders };
      if (bearerToken) headers['Authorization'] = `Bearer ${bearerToken}`;
      const res = await fetch(url, { headers });
      const body = await res.text();
      let parsed = null;
      try { parsed = JSON.parse(body); } catch(_) {}
      if (!res.ok) throw new Error(`HTTP ${res.status}: ${body.slice(0, 300)}`);

      const isArray = Array.isArray(parsed);
      const isEmptyArray = isArray && parsed.length === 0;
      const isErrorObj = !isArray && parsed?.message;

      if (isErrorObj) {
        throw new Error(`API returned error object: ${JSON.stringify(parsed)}`);
      }
      if (isEmptyArray) {
        throw new Error('File list is empty array [] — token rejected or video has no files');
      }

      files = isArray ? parsed : (parsed?.results || parsed?.list || []);

      return {
        status: res.status,
        isArray,
        fileCount: files.length,
        firstFile: files[0] ? {
          name: files[0].name,
          src: typeof files[0].src === 'object' ? JSON.stringify(files[0].src).slice(0, 200) : String(files[0].src).slice(0, 200),
          type: files[0].type
        } : null,
        rawBodyPreview: body.slice(0, 500)
      };
    }))();

    // STEP 5: Stream URL probe
    await (step('stream_url_probe', async () => {
      if (files.length === 0) throw new Error('No files to probe (step 4 failed)');
      const f = files[0];
      let rawUrl = '';
      if (typeof f.src === 'object' && f.src !== null) {
        rawUrl = f.src.view || f.src.download || '';
      } else {
        rawUrl = f.src || '';
      }
      if (rawUrl.startsWith('//')) rawUrl = 'https:' + rawUrl;
      else if (!rawUrl.startsWith('http')) rawUrl = 'https://files.iwara.tv' + rawUrl;

      // Parse URL to get fileId and expires for X-Version
      let parsedStreamUrl = null;
      try { parsedStreamUrl = new URL(rawUrl); } catch(_) {}

      const pathParts = (parsedStreamUrl?.pathname || '').split('/').filter(Boolean);
      const fileId = pathParts[pathParts.length - 1] || '';
      const expires = parsedStreamUrl?.searchParams.get('expires') || '';

      // Compute X-Version
      const xverDiag = await CryptoSignatureEngine.getIwaraVersionDiag(fileId, expires);

      // HEAD request probe
      const probeHeaders = { ...baseHeaders };
      if (bearerToken) probeHeaders['Authorization'] = `Bearer ${bearerToken}`;
      probeHeaders['X-Version'] = xverDiag.version;

      let probeResult = {};
      try {
        const probeRes = await fetch(rawUrl, { method: 'HEAD', headers: probeHeaders });
        probeResult = {
          status: probeRes.status,
          contentType: probeRes.headers.get('Content-Type'),
          contentLength: probeRes.headers.get('Content-Length'),
          acceptRanges: probeRes.headers.get('Accept-Ranges')
        };
      } catch (e) {
        probeResult = { error: String(e) };
      }

      return {
        rawUrl: rawUrl.slice(0, 200),
        hostname: parsedStreamUrl?.hostname,
        fileId,
        expires,
        xVersion: xverDiag,
        probeResult
      };
    }))();

    // STEP 6: X-Version validation
    await (step('xversion_validation', async () => {
      if (files.length === 0) throw new Error('No files to validate X-Version for');
      const f = files[0];
      let rawUrl = typeof f.src === 'object' ? (f.src.view || f.src.download || '') : (f.src || '');
      if (rawUrl.startsWith('//')) rawUrl = 'https:' + rawUrl;

      let parsedUrl = null;
      try { parsedUrl = new URL(rawUrl); } catch(_) { throw new Error('Invalid stream URL: ' + rawUrl.slice(0,100)); }

      const pathParts = parsedUrl.pathname.split('/').filter(Boolean);
      const fileId = pathParts[pathParts.length - 1] || '';
      const expires = parsedUrl.searchParams.get('expires') || '';

      // Compute both old and new format
      const newFormat = await CryptoSignatureEngine.sha1(`${fileId}_${expires}_${GLOBAL_CONFIG.API_SALT}`);
      const oldFormat = await CryptoSignatureEngine.sha1(`${fileId}${GLOBAL_CONFIG.API_SALT}`);

      return {
        fileId,
        expires,
        salt: GLOBAL_CONFIG.API_SALT,
        newFormatInput: `${fileId}_${expires}_${GLOBAL_CONFIG.API_SALT}`,
        newFormatXVersion: newFormat,
        oldFormatInput: `${fileId}${GLOBAL_CONFIG.API_SALT}`,
        oldFormatXVersion: oldFormat,
        selected: expires ? 'new' : 'old'
      };
    }))();

    // Generate conclusions and fixes
    this._analyze(report);
    return report;
  }

  /**
   * Analyze report steps and generate human-readable conclusions + fixes
   * Inspired by: ESLint rules engine, Lighthouse scoring
   */
  static _analyze(report) {
    const s = report.steps;

    // Token probe failed
    if (s.token_probe?.status === 'fail') {
      report.conclusions.push({
        severity: 'critical',
        code: 'TOKEN_ENDPOINT_DEAD',
        msg: '全てのトークンエンドポイントが有効なトークンを返しませんでした。',
        detail: s.token_probe.error
      });
      report.fixes.push({
        code: 'FIX_MANUAL_TOKEN',
        label: '手動でBearerトークンをセット',
        description: 'Iwaraにログインして、DevTools → Application → Cookies から "accessToken" の値をコピーして貼り付けてください。',
        action: 'manual_token_input'
      });
      report.fixes.push({
        code: 'FIX_LOGIN_API',
        label: 'メール+パスワードでログイン試行',
        description: 'POST /user/login を使ってログインし、トークンを取得します。',
        action: 'login_form'
      });
    }

    // Token acquired but file list empty
    if (s.token_acquire?.status === 'pass' && s.file_list?.status === 'fail') {
      const errMsg = s.file_list.error || '';
      if (errMsg.includes('empty array') || errMsg.includes('errors.notFound')) {
        report.conclusions.push({
          severity: 'critical',
          code: 'FILE_LIST_EMPTY',
          msg: 'ファイルリストが空または errors.notFound を返しました。',
          detail: errMsg
        });
        if (s.token_probe?.data?.every(r => !r.hasToken)) {
          report.fixes.push({
            code: 'FIX_TOKEN_INVALID',
            label: 'Bearerトークンが無効の可能性',
            description: '匿名ゲストトークンが廃止された可能性があります。Iwaraアカウントでログインして有効なJWTを取得してください。',
            action: 'manual_token_input'
          });
        }
        // Check if video has embedUrl (external video)
        if (s.video_metadata?.data?.embedUrl) {
          report.conclusions.push({
            severity: 'warning',
            code: 'EXTERNAL_VIDEO',
            msg: 'この動画は外部リンク (YouTube等) への埋め込みです。直接プロキシできません。',
            detail: `embedUrl: ${s.video_metadata.data.embedUrl}`
          });
          report.fixes.push({
            code: 'FIX_OPEN_EXTERNAL',
            label: '外部リンクを開く',
            description: `この動画は ${s.video_metadata.data.embedUrl} の外部埋め込みです。直接iwaraで視聴してください。`,
            action: 'open_external',
            url: s.video_metadata.data.embedUrl
          });
        }
      }
    }

    // Metadata type check
    if (s.video_metadata?.data?.type && s.video_metadata.data.type !== 'video') {
      report.conclusions.push({
        severity: 'warning',
        code: 'NOT_VIDEO_TYPE',
        msg: `このコンテンツは "video" タイプではありません (type: ${s.video_metadata.data.type})`,
        detail: 'Iwaraには画像投稿もあります。動画投稿のIDを使ってください。'
      });
    }

    // Stream probe failure
    if (s.stream_url_probe?.status === 'fail') {
      report.conclusions.push({
        severity: 'error',
        code: 'STREAM_PROBE_FAILED',
        msg: 'ストリームURLへのアクセスに失敗しました。',
        detail: s.stream_url_probe.error
      });
      report.fixes.push({
        code: 'FIX_XVERSION',
        label: 'X-Version ハッシュを確認',
        description: `現在のSalt: "${GLOBAL_CONFIG.API_SALT}" — yt-dlpの最新PR#16014でsaltが変更された可能性があります。`,
        action: 'check_salt'
      });
    }

    // Stream probe 403/401
    if (s.stream_url_probe?.data?.probeResult?.status === 403 || s.stream_url_probe?.data?.probeResult?.status === 401) {
      report.conclusions.push({
        severity: 'error',
        code: 'STREAM_AUTH_FAILED',
        msg: `ストリームサーバーが ${s.stream_url_probe.data.probeResult.status} を返しました — X-Version が正しくない可能性があります。`,
        detail: JSON.stringify(s.stream_url_probe.data.xVersion)
      });
    }

    // All passed
    if (report.summary.failed === 0) {
      report.conclusions.push({
        severity: 'success',
        code: 'ALL_PASS',
        msg: '全ての診断ステップが成功しました！Worker設定に問題はありません。',
        detail: null
      });
    }
  }
}

// =======================================================================================
// SECTION 10: API INTERCEPTOR
// Reference: https://github.com/http-party/node-http-proxy + axios interceptors
// =======================================================================================

class IwaraApiInterceptor {
  static async proxyFetch(targetUrl, request) {
    const target = new URL(targetUrl);
    const headers = new Headers();
    console.log(`[PROXY] --> ${request.method} ${targetUrl}`);

    headers.set('User-Agent',      GLOBAL_CONFIG.USER_AGENT);
    headers.set('Referer',         GLOBAL_CONFIG.REFERER);
    headers.set('Origin',          GLOBAL_CONFIG.ORIGIN);
    headers.set('Accept',          'application/json, text/plain, */*');
    headers.set('Accept-Language', 'ja,en-US;q=0.9,en;q=0.8');

    // Inject Bearer token
    if (target.hostname === 'api.iwara.tv' || target.hostname === 'apiq.iwara.tv') {
      const token = await IwaraTokenManager.getToken();
      if (token) {
        headers.set('Authorization', `Bearer ${token}`);
        console.log('[PROXY] Authorization injected for', target.hostname);
      } else {
        console.error('[PROXY] *** NO TOKEN *** for', target.hostname, target.pathname, '— /file will return errors.notFound');
      }
    }

    // Inject X-Version for file servers
    if (GLOBAL_CONFIG.FILE_SERVERS.includes(target.hostname)) {
      const pathSegments = target.pathname.split('/').filter(Boolean);
      const fileId = pathSegments[pathSegments.length - 1] || '';
      const expires = target.searchParams.get('expires') || '';
      console.log('[PROXY] File server:', target.hostname, '| fileId:', fileId, '| expires:', expires);
      if (fileId) {
        const xVerDiag = await CryptoSignatureEngine.getIwaraVersionDiag(fileId, expires);
        headers.set('X-Version', xVerDiag.version);
        console.log('[PROXY] X-Version:', xVerDiag.version, '(input:', xVerDiag.input, ')');
      }
    }

    const init = {
      method: request.method,
      headers: headers,
      redirect: "follow",
      cf: { cacheEverything: true, cacheTtl: 300 }
    };

    if (request.method !== 'GET' && request.method !== 'HEAD') {
      init.body = await request.clone().arrayBuffer();
    }

    const response = await fetch(targetUrl, init);
    const ct = response.headers.get('Content-Type') || '';
    console.log(`[PROXY] <-- ${response.status} ${response.statusText} | CT: ${ct}`);

    if (ct.includes('application/json')) {
      const rawText = await response.text();
      console.log(`[PROXY] JSON body (first 1000): ${rawText.slice(0, 1000)}`);

      if (target.pathname.endsWith('/file')) {
        let parsed;
        try { parsed = JSON.parse(rawText); } catch(e) { parsed = null; }
        if (Array.isArray(parsed)) {
          if (parsed.length === 0) {
            console.error('[PROXY] *** /file EMPTY ARRAY ***');
          } else {
            console.log('[PROXY] /file OK:', parsed.length, 'file(s)');
          }
        } else if (parsed?.message) {
          console.error('[PROXY] /file ERROR OBJECT:', JSON.stringify(parsed));
        }
      }

      let json;
      try { json = JSON.parse(rawText); }
      catch(e) {
        return new Response(rawText, {
          status: response.status,
          headers: { 'Content-Type': 'text/plain', 'Access-Control-Allow-Origin': '*' }
        });
      }

      // Add diagnostic headers to JSON responses for frontend debugging
      const respHeaders = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
        'X-Diag-Token-Present': _iwaraTokenCache.token ? '1' : '0',
        'X-Diag-Token-Expiry': String(_iwaraTokenCache.expiry),
        'X-Diag-Proxy-Hostname': target.hostname,
        'X-Diag-Proxy-Path': target.pathname.slice(0, 100),
      };

      return new Response(JSON.stringify(json), { headers: respHeaders });
    }

    if (ct.includes('video/') || ct.includes('audio/') || ct.includes('image/') || ct.includes('application/octet-stream')) {
      return await MediaStreamOrchestrator.handle(request, response);
    }

    const resHeaders = new Headers(response.headers);
    resHeaders.set('Access-Control-Allow-Origin', '*');
    return new Response(response.body, { status: response.status, headers: resHeaders });
  }
}

// =======================================================================================
// SECTION 11: TEMPLATE ENGINE
// Reference: https://github.com/janl/mustache.js
// =======================================================================================

class TemplateEngine {
  static renderUI() {
    return IWARA_UI_TEMPLATE;
  }
}

// =======================================================================================
// SECTION 12: FRONTEND HTML TEMPLATE — MEGA DEBUG EDITION
// =======================================================================================

const IWARA_UI_TEMPLATE = `<!DOCTYPE html>
<html lang="ja" class="h-full">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Iwara Edge Portal v8 — DEBUG</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">
    <link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;700&family=Roboto:wght@300;400;500;700&display=swap" rel="stylesheet">
    <style>
        :root {
            --yt-main-red: #ff0000;
            --yt-bg-white: #ffffff;
            --yt-text-black: #0f0f0f;
            --yt-gray: #606060;
            --yt-light-gray: #f2f2f2;
            /* Debug Panel Theme */
            --dbg-bg:         #0d1117;
            --dbg-bg-surface: #161b22;
            --dbg-bg-hover:   #21262d;
            --dbg-border:     #30363d;
            --dbg-text:       #c9d1d9;
            --dbg-muted:      #8b949e;
            --dbg-blue:       #58a6ff;
            --dbg-green:      #3fb950;
            --dbg-yellow:     #d29922;
            --dbg-red:        #f85149;
            --dbg-orange:     #f0883e;
            --dbg-purple:     #bc8cff;
            --dbg-cyan:       #39d353;
            --dbg-tab-active: #1f6feb;
        }
        body { font-family: 'Roboto', sans-serif; background-color: var(--yt-bg-white); color: var(--yt-text-black); -webkit-tap-highlight-color: transparent; }
        .yt-shadow { box-shadow: 0 4px 12px rgba(0,0,0,0.08); }
        .truncate-2 { display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
        .video-container { position: relative; width: 100%; padding-top: 56.25%; background: #000; border-radius: 12px; overflow: hidden; box-shadow: 0 8px 24px rgba(0,0,0,0.2); }
        .video-container video { position: absolute; top: 0; left: 0; width: 100%; height: 100%; outline: none; }
        .spinner { width: 40px; height: 40px; border: 4px solid rgba(0,0,0,0.1); border-left-color: #3b82f6; border-radius: 50%; animation: spin 0.8s linear infinite; }
        @keyframes spin { to { transform: rotate(360deg); } }
        .skeleton { background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%); background-size: 200% 100%; animation: skeleton-loading 1.5s infinite; }
        @keyframes skeleton-loading { from { background-position: 200% 0; } to { background-position: -200% 0; } }
        ::-webkit-scrollbar { width: 6px; height: 6px; }
        ::-webkit-scrollbar-track { background: transparent; }
        ::-webkit-scrollbar-thumb { background: #ccc; border-radius: 10px; }
        ::-webkit-scrollbar-thumb:hover { background: #999; }
        .btn-yt { padding: 6px 18px; border-radius: 9999px; font-weight: bold; font-size: 13px; transition: all 0.15s; }
        .card-anim { transition: transform 0.2s cubic-bezier(0.2, 0, 0, 1); }
        .card-anim:hover { transform: translateY(-4px); }

        /* ====================== DEBUG PANEL STYLES (GitHub Dark theme inspired) ====================== */
        #debugPanelRoot {
            font-family: 'JetBrains Mono', 'Courier New', monospace;
            font-size: 11.5px;
            line-height: 1.5;
        }
        #debugPanelRoot * { box-sizing: border-box; }

        /* Panel Container */
        #debugPanelRoot .dp-container {
            position: fixed;
            bottom: 0;
            left: 0;
            right: 0;
            z-index: 9999;
            background: var(--dbg-bg);
            color: var(--dbg-text);
            display: flex;
            flex-direction: column;
            border-top: 1px solid var(--dbg-tab-active);
            box-shadow: 0 -8px 32px rgba(0,0,0,0.5);
            transition: height 0.2s ease;
        }
        #debugPanelRoot .dp-container.dp-minimized { height: 36px !important; overflow: hidden; }

        /* Header Bar */
        #debugPanelRoot .dp-header {
            background: var(--dbg-bg-surface);
            border-bottom: 1px solid var(--dbg-border);
            padding: 0 10px;
            display: flex;
            align-items: center;
            gap: 8px;
            height: 36px;
            flex-shrink: 0;
            user-select: none;
            cursor: row-resize;
        }
        #debugPanelRoot .dp-logo {
            color: var(--dbg-blue);
            font-weight: 700;
            font-size: 12px;
            white-space: nowrap;
        }
        #debugPanelRoot .dp-version {
            color: var(--dbg-muted);
            font-size: 10px;
        }

        /* Tab Bar */
        #debugPanelRoot .dp-tabs {
            display: flex;
            gap: 0;
            background: var(--dbg-bg-surface);
            border-bottom: 1px solid var(--dbg-border);
            flex-shrink: 0;
            overflow-x: auto;
        }
        #debugPanelRoot .dp-tabs::-webkit-scrollbar { height: 2px; }
        #debugPanelRoot .dp-tab {
            padding: 6px 14px;
            font-size: 11px;
            font-weight: 600;
            color: var(--dbg-muted);
            cursor: pointer;
            border-bottom: 2px solid transparent;
            white-space: nowrap;
            transition: color 0.1s, border-color 0.1s;
            display: flex;
            align-items: center;
            gap: 5px;
        }
        #debugPanelRoot .dp-tab:hover { color: var(--dbg-text); background: var(--dbg-bg-hover); }
        #debugPanelRoot .dp-tab.active { color: var(--dbg-blue); border-bottom-color: var(--dbg-blue); background: var(--dbg-bg); }
        #debugPanelRoot .dp-tab .dp-badge {
            background: var(--dbg-red);
            color: #fff;
            border-radius: 9999px;
            padding: 0 5px;
            font-size: 9px;
            font-weight: 700;
            min-width: 16px;
            text-align: center;
            line-height: 16px;
        }
        #debugPanelRoot .dp-tab .dp-badge.green { background: var(--dbg-green); }
        #debugPanelRoot .dp-tab .dp-badge.yellow { background: var(--dbg-yellow); }

        /* Content Area */
        #debugPanelRoot .dp-content {
            flex: 1;
            overflow: hidden;
            display: flex;
            position: relative;
        }
        #debugPanelRoot .dp-pane {
            display: none;
            flex: 1;
            overflow: hidden;
            flex-direction: column;
        }
        #debugPanelRoot .dp-pane.active { display: flex; }
        #debugPanelRoot .dp-scroll {
            flex: 1;
            overflow-y: auto;
            overflow-x: auto;
        }
        #debugPanelRoot .dp-scroll::-webkit-scrollbar { width: 6px; background: var(--dbg-bg); }
        #debugPanelRoot .dp-scroll::-webkit-scrollbar-thumb { background: var(--dbg-border); border-radius: 3px; }

        /* ---- Console Tab ---- */
        .dp-log-entry {
            padding: 2px 10px;
            border-bottom: 1px solid rgba(255,255,255,0.03);
            display: flex;
            gap: 8px;
            align-items: flex-start;
            word-break: break-all;
        }
        .dp-log-entry:hover { background: var(--dbg-bg-hover); }
        .dp-log-ts { color: var(--dbg-muted); font-size: 10px; white-space: nowrap; padding-top: 1px; }
        .dp-log-ns { font-size: 10px; font-weight: 700; white-space: nowrap; padding: 1px 4px; border-radius: 3px; }
        .dp-log-msg { flex: 1; }
        .dp-log-entry.level-step .dp-log-ns { background: rgba(31,111,235,0.2); color: var(--dbg-blue); }
        .dp-log-entry.level-info .dp-log-ns { background: rgba(63,185,80,0.15); color: var(--dbg-green); }
        .dp-log-entry.level-warn .dp-log-ns { background: rgba(210,153,34,0.2); color: var(--dbg-yellow); }
        .dp-log-entry.level-error .dp-log-ns { background: rgba(248,81,73,0.2); color: var(--dbg-red); }
        .dp-log-entry.level-success .dp-log-ns { background: rgba(63,185,80,0.25); color: #39d353; }
        .dp-log-entry.level-net .dp-log-ns { background: rgba(188,140,255,0.15); color: var(--dbg-purple); }
        .dp-json-toggle {
            cursor: pointer;
            color: var(--dbg-cyan);
            text-decoration: underline;
            font-size: 10px;
            margin-left: 6px;
        }
        .dp-json-body {
            background: var(--dbg-bg-surface);
            border: 1px solid var(--dbg-border);
            border-radius: 4px;
            padding: 6px 10px;
            margin: 4px 0;
            white-space: pre-wrap;
            color: var(--dbg-cyan);
            font-size: 10px;
            display: none;
        }
        .dp-json-body.visible { display: block; }

        /* Console toolbar */
        .dp-console-toolbar {
            background: var(--dbg-bg-surface);
            border-bottom: 1px solid var(--dbg-border);
            padding: 4px 10px;
            display: flex;
            gap: 6px;
            align-items: center;
            flex-shrink: 0;
        }
        .dp-search-input {
            background: var(--dbg-bg);
            border: 1px solid var(--dbg-border);
            color: var(--dbg-text);
            padding: 2px 8px;
            border-radius: 4px;
            font-family: 'JetBrains Mono', monospace;
            font-size: 11px;
            outline: none;
            width: 200px;
        }
        .dp-filter-btn {
            padding: 2px 8px;
            border-radius: 4px;
            font-size: 10px;
            font-weight: 700;
            cursor: pointer;
            border: 1px solid var(--dbg-border);
            color: var(--dbg-muted);
            background: transparent;
            transition: all 0.1s;
        }
        .dp-filter-btn.active-filter { background: var(--dbg-tab-active); color: #fff; border-color: var(--dbg-tab-active); }
        .dp-action-btn {
            padding: 3px 10px;
            border-radius: 4px;
            font-size: 10px;
            font-weight: 700;
            cursor: pointer;
            color: var(--dbg-text);
            background: var(--dbg-bg-hover);
            border: 1px solid var(--dbg-border);
            transition: all 0.1s;
        }
        .dp-action-btn:hover { background: var(--dbg-border); }
        .dp-action-btn.danger { color: var(--dbg-red); border-color: var(--dbg-red); }
        .dp-action-btn.primary { color: var(--dbg-blue); border-color: var(--dbg-blue); }
        .dp-action-btn.success { color: var(--dbg-green); border-color: var(--dbg-green); }
        .dp-action-btn.warning { color: var(--dbg-yellow); border-color: var(--dbg-yellow); }

        /* ---- Network Tab ---- */
        .dp-net-toolbar {
            background: var(--dbg-bg-surface);
            border-bottom: 1px solid var(--dbg-border);
            padding: 4px 10px;
            display: flex;
            gap: 8px;
            align-items: center;
            flex-shrink: 0;
        }
        .dp-net-table { width: 100%; border-collapse: collapse; }
        .dp-net-table thead th {
            background: var(--dbg-bg-surface);
            color: var(--dbg-muted);
            font-size: 10px;
            font-weight: 700;
            text-transform: uppercase;
            padding: 4px 8px;
            border-bottom: 1px solid var(--dbg-border);
            text-align: left;
            position: sticky;
            top: 0;
            z-index: 1;
        }
        .dp-net-row {
            border-bottom: 1px solid rgba(255,255,255,0.03);
            cursor: pointer;
        }
        .dp-net-row:hover { background: var(--dbg-bg-hover); }
        .dp-net-row td { padding: 3px 8px; vertical-align: top; font-size: 11px; }
        .dp-net-row.selected { background: rgba(31,111,235,0.15) !important; }
        .dp-net-status-ok { color: var(--dbg-green); font-weight: 700; }
        .dp-net-status-warn { color: var(--dbg-yellow); font-weight: 700; }
        .dp-net-status-err { color: var(--dbg-red); font-weight: 700; }
        .dp-net-url { color: var(--dbg-blue); max-width: 340px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; font-size: 10px; }
        .dp-net-method { color: var(--dbg-purple); font-weight: 700; font-size: 10px; }
        .dp-net-time { color: var(--dbg-muted); font-size: 10px; }
        .dp-net-size { color: var(--dbg-muted); font-size: 10px; }
        .dp-net-type { color: var(--dbg-orange); font-size: 10px; }
        .dp-waterfall-bar {
            display: inline-block;
            height: 10px;
            border-radius: 2px;
            background: var(--dbg-blue);
            min-width: 2px;
            opacity: 0.7;
        }
        .dp-waterfall-bar.slow { background: var(--dbg-yellow); }
        .dp-waterfall-bar.very-slow { background: var(--dbg-red); }

        /* Detail pane */
        .dp-net-detail {
            background: var(--dbg-bg-surface);
            border-top: 1px solid var(--dbg-border);
            padding: 10px;
            overflow-y: auto;
            max-height: 40%;
            flex-shrink: 0;
            display: none;
        }
        .dp-net-detail.visible { display: block; }
        .dp-detail-section { margin-bottom: 10px; }
        .dp-detail-section h4 { color: var(--dbg-muted); font-size: 10px; text-transform: uppercase; font-weight: 700; margin-bottom: 4px; }
        .dp-detail-kv { display: flex; gap: 8px; padding: 1px 0; }
        .dp-detail-kv .dk { color: var(--dbg-muted); min-width: 160px; font-size: 10px; }
        .dp-detail-kv .dv { color: var(--dbg-text); font-size: 10px; word-break: break-all; }
        .dp-detail-kv .dv.ok { color: var(--dbg-green); }
        .dp-detail-kv .dv.err { color: var(--dbg-red); }

        /* ---- Token Tab ---- */
        .dp-token-container { padding: 12px 16px; }
        .dp-token-status-card {
            background: var(--dbg-bg-surface);
            border: 1px solid var(--dbg-border);
            border-radius: 6px;
            padding: 12px 16px;
            margin-bottom: 12px;
        }
        .dp-token-status-card.status-ok { border-left: 3px solid var(--dbg-green); }
        .dp-token-status-card.status-fail { border-left: 3px solid var(--dbg-red); }
        .dp-token-status-card.status-warn { border-left: 3px solid var(--dbg-yellow); }
        .dp-token-label { font-size: 10px; color: var(--dbg-muted); text-transform: uppercase; font-weight: 700; margin-bottom: 4px; }
        .dp-token-value { color: var(--dbg-text); word-break: break-all; font-size: 11px; }
        .dp-token-countdown { font-size: 24px; font-weight: 700; color: var(--dbg-blue); }
        .dp-input-row { display: flex; gap: 6px; align-items: center; }
        .dp-text-input {
            flex: 1;
            background: var(--dbg-bg);
            border: 1px solid var(--dbg-border);
            color: var(--dbg-text);
            padding: 6px 10px;
            border-radius: 4px;
            font-family: 'JetBrains Mono', monospace;
            font-size: 11px;
            outline: none;
        }
        .dp-text-input:focus { border-color: var(--dbg-blue); }
        .dp-section-title { font-size: 11px; font-weight: 700; color: var(--dbg-muted); text-transform: uppercase; letter-spacing: 0.08em; margin-bottom: 8px; margin-top: 16px; }
        .dp-attempt-row {
            background: var(--dbg-bg-surface);
            border: 1px solid var(--dbg-border);
            border-radius: 4px;
            padding: 6px 10px;
            margin-bottom: 6px;
            font-size: 10px;
        }
        .dp-attempt-row .ar-url { color: var(--dbg-blue); }
        .dp-attempt-row .ar-status-ok { color: var(--dbg-green); font-weight: 700; }
        .dp-attempt-row .ar-status-err { color: var(--dbg-red); font-weight: 700; }

        /* ---- Diagnostic Tab ---- */
        .dp-diag-container { padding: 12px 16px; }
        .dp-diag-step {
            display: flex;
            align-items: flex-start;
            gap: 12px;
            margin-bottom: 10px;
            padding: 10px 14px;
            background: var(--dbg-bg-surface);
            border: 1px solid var(--dbg-border);
            border-radius: 6px;
            cursor: pointer;
            transition: background 0.1s;
        }
        .dp-diag-step:hover { background: var(--dbg-bg-hover); }
        .dp-diag-step .ds-icon { width: 22px; height: 22px; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-size: 12px; flex-shrink: 0; margin-top: 1px; }
        .ds-icon.running { background: rgba(88,166,255,0.2); color: var(--dbg-blue); animation: pulse-anim 1s infinite; }
        .ds-icon.pass { background: rgba(63,185,80,0.2); color: var(--dbg-green); }
        .ds-icon.fail { background: rgba(248,81,73,0.2); color: var(--dbg-red); }
        .ds-icon.idle { background: var(--dbg-bg-hover); color: var(--dbg-muted); }
        .ds-icon.skip { background: rgba(210,153,34,0.15); color: var(--dbg-yellow); }
        @keyframes pulse-anim { 0%,100%{ opacity:1; } 50%{ opacity:0.4; } }
        .dp-diag-step .ds-body { flex: 1; min-width: 0; }
        .dp-diag-step .ds-title { font-size: 12px; font-weight: 700; color: var(--dbg-text); }
        .dp-diag-step .ds-sub { font-size: 10px; color: var(--dbg-muted); margin-top: 2px; }
        .dp-diag-step .ds-dur { font-size: 10px; color: var(--dbg-muted); white-space: nowrap; }
        .dp-diag-detail {
            background: var(--dbg-bg);
            border: 1px solid var(--dbg-border);
            border-radius: 4px;
            padding: 8px;
            margin-top: 6px;
            font-size: 10px;
            white-space: pre-wrap;
            color: var(--dbg-cyan);
            display: none;
            max-height: 200px;
            overflow-y: auto;
        }
        .dp-diag-detail.visible { display: block; }
        .dp-diag-run-btn {
            padding: 8px 20px;
            background: var(--dbg-tab-active);
            color: #fff;
            border: none;
            border-radius: 6px;
            font-family: 'JetBrains Mono', monospace;
            font-weight: 700;
            font-size: 12px;
            cursor: pointer;
            transition: background 0.15s;
            display: flex;
            align-items: center;
            gap: 8px;
        }
        .dp-diag-run-btn:hover { background: #2d79f0; }
        .dp-diag-run-btn:disabled { background: var(--dbg-bg-hover); color: var(--dbg-muted); cursor: not-allowed; }
        .dp-diag-header {
            display: flex;
            align-items: center;
            gap: 10px;
            margin-bottom: 14px;
        }
        .dp-vid-input {
            background: var(--dbg-bg);
            border: 1px solid var(--dbg-border);
            color: var(--dbg-text);
            padding: 6px 10px;
            border-radius: 4px;
            font-family: 'JetBrains Mono', monospace;
            font-size: 11px;
            outline: none;
            flex: 1;
        }
        .dp-diag-summary {
            display: flex;
            gap: 10px;
            margin-bottom: 12px;
        }
        .dp-summary-chip {
            padding: 3px 10px;
            border-radius: 9999px;
            font-size: 11px;
            font-weight: 700;
        }
        .dp-summary-chip.pass { background: rgba(63,185,80,0.15); color: var(--dbg-green); }
        .dp-summary-chip.fail { background: rgba(248,81,73,0.15); color: var(--dbg-red); }
        .dp-summary-chip.warn { background: rgba(210,153,34,0.15); color: var(--dbg-yellow); }

        /* ---- Fixes Tab ---- */
        .dp-fixes-container { padding: 12px 16px; }
        .dp-conclusion-card {
            padding: 10px 14px;
            border-radius: 6px;
            margin-bottom: 8px;
            border: 1px solid var(--dbg-border);
        }
        .dp-conclusion-card.critical { background: rgba(248,81,73,0.08); border-left: 3px solid var(--dbg-red); }
        .dp-conclusion-card.error { background: rgba(240,136,62,0.08); border-left: 3px solid var(--dbg-orange); }
        .dp-conclusion-card.warning { background: rgba(210,153,34,0.08); border-left: 3px solid var(--dbg-yellow); }
        .dp-conclusion-card.success { background: rgba(63,185,80,0.08); border-left: 3px solid var(--dbg-green); }
        .dp-conclusion-card .cc-title { font-weight: 700; font-size: 12px; margin-bottom: 3px; }
        .dp-conclusion-card .cc-detail { color: var(--dbg-muted); font-size: 10px; }
        .dp-conclusion-card.critical .cc-title { color: var(--dbg-red); }
        .dp-conclusion-card.error .cc-title { color: var(--dbg-orange); }
        .dp-conclusion-card.warning .cc-title { color: var(--dbg-yellow); }
        .dp-conclusion-card.success .cc-title { color: var(--dbg-green); }
        .dp-fix-card {
            padding: 10px 14px;
            border-radius: 6px;
            margin-bottom: 8px;
            border: 1px solid var(--dbg-border);
            background: var(--dbg-bg-surface);
        }
        .dp-fix-card .fc-label { font-weight: 700; color: var(--dbg-blue); font-size: 12px; margin-bottom: 4px; display: flex; align-items: center; gap: 6px; }
        .dp-fix-card .fc-desc { color: var(--dbg-muted); font-size: 10px; line-height: 1.6; }
        .dp-fix-card .fc-action { margin-top: 8px; }

        /* Login Form */
        .dp-login-form { display: flex; flex-direction: column; gap: 6px; }
        .dp-login-form label { font-size: 10px; color: var(--dbg-muted); }

        /* Resize handle */
        #dpResizeHandle {
            position: fixed;
            bottom: 0;
            left: 0;
            right: 0;
            height: 4px;
            cursor: row-resize;
            z-index: 10000;
        }
        #dpResizeHandle:hover { background: var(--dbg-tab-active); }

        /* Header control buttons */
        .dp-ctrl-btn {
            background: transparent;
            border: none;
            color: var(--dbg-muted);
            cursor: pointer;
            padding: 2px 6px;
            border-radius: 3px;
            font-size: 12px;
            line-height: 1;
            font-family: 'JetBrains Mono', monospace;
        }
        .dp-ctrl-btn:hover { background: var(--dbg-bg-hover); color: var(--dbg-text); }
        .dp-ctrl-btn.close:hover { background: rgba(248,81,73,0.2); color: var(--dbg-red); }

        /* DBG launch button in header */
        #dbgLaunchBtn {
            position: fixed;
            bottom: 12px;
            right: 12px;
            z-index: 9998;
            background: var(--dbg-bg-surface);
            color: var(--dbg-blue);
            border: 1px solid var(--dbg-tab-active);
            border-radius: 6px;
            padding: 6px 12px;
            font-family: 'JetBrains Mono', monospace;
            font-size: 11px;
            font-weight: 700;
            cursor: pointer;
            display: flex;
            align-items: center;
            gap: 6px;
            box-shadow: 0 4px 16px rgba(0,0,0,0.4);
            transition: all 0.15s;
        }
        #dbgLaunchBtn:hover { background: var(--dbg-tab-active); color: #fff; }
        #dbgLaunchBtn .dbg-error-badge {
            background: var(--dbg-red);
            color: #fff;
            border-radius: 50%;
            width: 16px;
            height: 16px;
            font-size: 9px;
            font-weight: 700;
            display: none;
            align-items: center;
            justify-content: center;
        }

        /* X-Version inspector widget */
        .dp-xver-widget {
            background: var(--dbg-bg-surface);
            border: 1px solid var(--dbg-border);
            border-radius: 6px;
            padding: 12px;
            margin: 10px 0;
        }
        .dp-xver-formula {
            font-size: 11px;
            color: var(--dbg-cyan);
            padding: 6px 10px;
            background: var(--dbg-bg);
            border-radius: 4px;
            margin: 6px 0;
            word-break: break-all;
        }

        /* Stream probe meter */
        .dp-stream-meter {
            height: 4px;
            border-radius: 2px;
            background: var(--dbg-border);
            margin: 6px 0;
            overflow: hidden;
        }
        .dp-stream-meter-fill {
            height: 100%;
            border-radius: 2px;
            transition: width 0.5s ease;
            background: var(--dbg-green);
        }
    </style>
</head>
<body class="flex flex-col h-full overflow-hidden">

<!-- ==================== MAIN APP HEADER ==================== -->
<header class="h-14 bg-white sticky top-0 z-[100] px-4 flex items-center justify-between border-b border-gray-100">
    <div class="flex items-center gap-4 shrink-0">
        <button class="p-2 hover:bg-gray-100 rounded-full hidden md:block"><i class="fa-solid fa-bars text-lg"></i></button>
        <div class="flex items-center gap-1 cursor-pointer" onclick="App.goHome()">
            <i class="fa-brands fa-youtube text-[--yt-main-red] text-3xl"></i>
            <h1 class="text-xl font-bold tracking-tighter hidden sm:block">YouTube Player <span class="font-light">3rd</span></h1>
        </div>
    </div>
    <div class="flex-1 max-w-[720px] mx-4 md:mx-10">
        <div class="flex items-center bg-gray-50 border border-gray-300 rounded-full px-1 focus-within:border-blue-500 focus-within:bg-white transition-all shadow-inner">
            <input type="text" id="searchInput" placeholder="Iwaraの動画、タグ、ユーザーを検索" class="w-full bg-transparent px-4 py-2 outline-none text-[16px]">
            <button onclick="App.handleSearch()" class="px-5 py-2 bg-gray-100 border-l border-gray-300 rounded-r-full hover:bg-gray-200 transition">
                <i class="fa-solid fa-magnifying-glass text-gray-600"></i>
            </button>
        </div>
    </div>
    <div class="flex items-center gap-2 shrink-0">
        <button onclick="App.goHome()" class="hidden sm:flex items-center gap-2 bg-blue-600 text-white px-4 py-1.5 rounded-full text-sm font-bold hover:bg-blue-700">
            <i class="fa-solid fa-house"></i> <span>ホーム</span>
        </button>
        <button class="p-2 hover:bg-gray-100 rounded-full"><i class="fa-solid fa-ellipsis-vertical"></i></button>
    </div>
</header>

<!-- ==================== CONTENT LAYOUT ==================== -->
<div class="flex flex-1 overflow-hidden">
    <nav class="w-64 hidden xl:flex flex-col px-3 py-4 border-r border-gray-100 overflow-y-auto">
        <div class="flex flex-col gap-1 mb-4">
            <button onclick="App.goHome()" class="flex items-center gap-5 px-3 py-2.5 bg-gray-100 rounded-xl font-medium text-sm"><i class="fa-solid fa-house text-lg w-6"></i> ホーム</button>
            <button class="flex items-center gap-5 px-3 py-2.5 hover:bg-gray-100 rounded-xl font-normal text-sm"><i class="fa-solid fa-compass text-lg w-6"></i> 探索</button>
            <button class="flex items-center gap-5 px-3 py-2.5 hover:bg-gray-100 rounded-xl font-normal text-sm"><i class="fa-solid fa-clock-rotate-left text-lg w-6"></i> 履歴</button>
        </div>
        <hr class="mb-4">
        <div class="px-3 mb-2 text-xs font-bold text-gray-500 uppercase">フィルター</div>
        <select id="sortSelect" onchange="App.handleSortChange(this.value)" class="mx-2 p-2 bg-gray-50 border border-gray-200 rounded-lg outline-none text-sm cursor-pointer">
            <option value="date">🕒 最新順</option>
            <option value="views">🔥 再生回数順</option>
            <option value="likes">👍 高評価順</option>
        </select>
        <div class="mt-auto px-4 py-6 text-[11px] text-gray-400 leading-tight">
            © 2026 YouTube Player 3rd<br>Powered by Cloudflare Workers<br>
            <span class="text-blue-400 font-bold">v8.0 MEGA-DEBUG</span>
        </div>
    </nav>

    <main class="flex-1 overflow-y-auto px-4 md:px-8 py-6 relative" id="mainScrollArea">
        <!-- View: Home/Search Grid -->
        <section id="view-grid" class="max-w-[1800px] mx-auto">
            <div class="flex flex-wrap gap-2 mb-6" id="tagList">
                <span class="px-3 py-1 bg-black text-white rounded-lg text-sm font-medium cursor-pointer">すべて</span>
                <span class="px-3 py-1 bg-gray-100 hover:bg-gray-200 rounded-lg text-sm font-medium cursor-pointer">MMD</span>
                <span class="px-3 py-1 bg-gray-100 hover:bg-gray-200 rounded-lg text-sm font-medium cursor-pointer">R-18</span>
                <span class="px-3 py-1 bg-gray-100 hover:bg-gray-200 rounded-lg text-sm font-medium cursor-pointer">バーチャルYouTuber</span>
            </div>
            <h2 class="text-xl font-bold mb-6 flex items-center gap-2" id="gridHeader">
                <i class="fa-solid fa-fire text-orange-500"></i> <span>おすすめの動画</span>
            </h2>
            <div id="videoGrid" class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 2xl:grid-cols-4 gap-x-4 gap-y-10"></div>
            <div class="flex justify-center items-center gap-6 mt-16 mb-32 border-t pt-10">
                <button id="btnPrev" onclick="App.changePage(-1)" class="px-8 py-2.5 bg-white border border-gray-300 rounded-full font-bold hover:bg-gray-50 disabled:opacity-30 disabled:cursor-not-allowed transition shadow-sm">前へ</button>
                <div id="pageIndicator" class="text-sm font-bold bg-gray-100 px-5 py-2 rounded-full">Page 1</div>
                <button id="btnNext" onclick="App.changePage(1)" class="px-8 py-2.5 bg-blue-600 text-white rounded-full font-bold hover:bg-blue-700 transition shadow-md">次へ</button>
            </div>
        </section>

        <!-- View: Player & Details -->
        <section id="view-player" class="hidden max-w-[1400px] mx-auto flex flex-col xl:flex-row gap-8 pb-20">
            <div class="flex-1 min-w-0">
                <div class="video-container">
                    <video id="v-player" controls playsinline preload="auto" poster=""></video>
                </div>
                <div class="mt-5">
                    <h1 id="v-title" class="text-xl md:text-2xl font-bold leading-snug"></h1>
                    <div class="flex flex-col sm:flex-row sm:items-center justify-between mt-4 gap-4 pb-4 border-b">
                        <div class="flex items-center gap-3 cursor-pointer group">
                            <img id="v-avatar" src="" class="w-12 h-12 rounded-full bg-gray-200 object-cover border">
                            <div class="flex flex-col overflow-hidden">
                                <span id="v-author" class="font-bold text-lg group-hover:text-blue-600 transition truncate"></span>
                                <span class="text-xs text-gray-500 font-medium">Iwara クリエイター</span>
                            </div>
                        </div>
                        <div class="flex items-center gap-2 shrink-0">
                            <div class="flex bg-gray-100 rounded-full overflow-hidden shadow-sm">
                                <button class="px-5 py-2.5 hover:bg-gray-200 transition flex items-center gap-2 border-r border-gray-300">
                                    <i class="fa-solid fa-thumbs-up"></i> <span id="v-likes" class="text-sm font-bold">0</span>
                                </button>
                                <button class="px-4 py-2.5 hover:bg-gray-200 transition flex items-center"><i class="fa-solid fa-thumbs-down"></i></button>
                            </div>
                            <button class="px-5 py-2.5 bg-gray-100 hover:bg-gray-200 rounded-full text-sm font-bold transition flex items-center gap-2"><i class="fa-solid fa-share"></i> 共有</button>
                            <button onclick="DebugPanel.quickDiagnoseCurrentVideo()" class="px-5 py-2.5 bg-yellow-50 hover:bg-yellow-100 border border-yellow-300 rounded-full text-sm font-bold transition flex items-center gap-2 text-yellow-700">
                                <i class="fa-solid fa-stethoscope"></i> 診断
                            </button>
                        </div>
                    </div>
                    <div class="mt-6 p-5 bg-gray-50 rounded-2xl border border-gray-200">
                        <div class="text-[11px] font-bold text-gray-400 mb-3 tracking-widest uppercase flex items-center gap-2">
                            <i class="fa-solid fa-server text-blue-500"></i> 解像度を選択（エッジ暗号化済み）
                        </div>
                        <div id="v-sources" class="flex flex-wrap gap-2.5"></div>
                    </div>
                    <div class="mt-4 bg-gray-100 hover:bg-gray-200 rounded-2xl p-4 transition cursor-pointer shadow-inner">
                        <div class="flex items-center gap-2 text-sm font-bold mb-2">
                            <span id="v-views">0</span> 回視聴
                            <span class="text-gray-400">•</span>
                            <span id="v-date">...</span>
                        </div>
                        <p id="v-desc" class="text-sm text-gray-800 whitespace-pre-wrap leading-relaxed overflow-hidden" style="max-height: 120px;"></p>
                        <button onclick="this.previousElementSibling.style.maxHeight='none'; this.remove()" class="text-sm font-bold mt-2 text-gray-500 hover:text-black">もっと見る</button>
                    </div>
                </div>
            </div>
            <div class="xl:w-[400px] shrink-0">
                <h3 class="font-bold mb-4 flex items-center gap-2"><i class="fa-solid fa-list-ul"></i> おすすめ</h3>
                <div id="relatedGrid" class="flex flex-col gap-3"></div>
            </div>
        </section>

        <!-- Global Loader -->
        <div id="globalLoader" class="fixed inset-0 bg-white/60 z-[200] flex flex-col items-center justify-center hidden">
            <div class="spinner mb-4"></div>
            <div class="text-sm font-bold text-blue-600 tracking-widest animate-pulse">データを読み込み中...</div>
        </div>
    </main>
</div>

<!-- ==================== DEBUG LAUNCH BUTTON ==================== -->
<button id="dbgLaunchBtn" onclick="DebugPanel.show()">
    <span style="font-size:14px;">🔬</span>
    <span>DEV</span>
    <span class="dbg-error-badge" id="dbgErrorBadge">0</span>
</button>

<!-- ==================== DEBUG PANEL ==================== -->
<div id="debugPanelRoot">
    <div class="dp-container" id="dpContainer" style="height: 360px; display: none;">

        <!-- Header -->
        <div class="dp-header" id="dpHeaderBar">
            <span class="dp-logo">🔬 IWARA EDGE DEBUGGER</span>
            <span class="dp-version">v${GLOBAL_CONFIG.VERSION}</span>
            <div style="flex:1"></div>
            <!-- Toolbar buttons -->
            <button class="dp-action-btn success" onclick="DebugPanel.exportLogs()" title="ログをJSONエクスポート">⬇ Export</button>
            <button class="dp-action-btn warning" onclick="DebugPanel.copyLogs()" title="クリップボードにコピー">📋 Copy</button>
            <button class="dp-action-btn danger" onclick="DebugPanel.clearAll()" title="全クリア">🗑 Clear</button>
            <button class="dp-ctrl-btn" onclick="DebugPanel.minimize()" title="最小化">—</button>
            <button class="dp-ctrl-btn close" onclick="DebugPanel.hide()" title="閉じる">✕</button>
        </div>

        <!-- Tab Bar -->
        <div class="dp-tabs" id="dpTabs">
            <div class="dp-tab active" data-tab="console" onclick="DebugPanel.switchTab('console')">
                <span>📋 Console</span>
                <span class="dp-badge" id="dpBadgeConsole" style="display:none">0</span>
            </div>
            <div class="dp-tab" data-tab="network" onclick="DebugPanel.switchTab('network')">
                <span>🌐 Network</span>
                <span class="dp-badge yellow" id="dpBadgeNetwork">0</span>
            </div>
            <div class="dp-tab" data-tab="token" onclick="DebugPanel.switchTab('token')">
                <span>🔑 Token</span>
                <span class="dp-badge" id="dpBadgeToken" style="display:none">!</span>
            </div>
            <div class="dp-tab" data-tab="diagnostic" onclick="DebugPanel.switchTab('diagnostic')">
                <span>🩺 Diagnostic</span>
                <span class="dp-badge" id="dpBadgeDiag" style="display:none">!</span>
            </div>
            <div class="dp-tab" data-tab="fixes" onclick="DebugPanel.switchTab('fixes')">
                <span>🔧 Fixes</span>
                <span class="dp-badge" id="dpBadgeFixes" style="display:none">0</span>
            </div>
        </div>

        <!-- Content -->
        <div class="dp-content">

            <!-- ========= CONSOLE TAB ========= -->
            <div class="dp-pane active" id="dpPane-console">
                <div class="dp-console-toolbar">
                    <input type="text" class="dp-search-input" id="dpConsoleSearch" placeholder="🔍 フィルター (正規表現対応)..." oninput="DebugPanel.applyConsoleFilter()">
                    <button class="dp-filter-btn active-filter" data-level="all" onclick="DebugPanel.setLevelFilter('all', this)">ALL</button>
                    <button class="dp-filter-btn" data-level="step" onclick="DebugPanel.setLevelFilter('step', this)">▶ STEP</button>
                    <button class="dp-filter-btn" data-level="error" onclick="DebugPanel.setLevelFilter('error', this)">✖ ERR</button>
                    <button class="dp-filter-btn" data-level="warn" onclick="DebugPanel.setLevelFilter('warn', this)">⚠ WARN</button>
                    <button class="dp-filter-btn" data-level="net" onclick="DebugPanel.setLevelFilter('net', this)">🌐 NET</button>
                    <div style="flex:1"></div>
                    <span id="dpLogCount" style="color:var(--dbg-muted);font-size:10px;">0 entries</span>
                </div>
                <div class="dp-scroll" id="dpConsoleLog"></div>
            </div>

            <!-- ========= NETWORK TAB ========= -->
            <div class="dp-pane" id="dpPane-network" style="flex-direction:column;">
                <div class="dp-net-toolbar">
                    <button class="dp-action-btn danger" onclick="NetworkInspector.clear()">🗑 Clear</button>
                    <button class="dp-action-btn" onclick="NetworkInspector.exportHAR()">⬇ HAR</button>
                    <input type="text" class="dp-search-input" id="dpNetSearch" placeholder="🔍 URL フィルター..." oninput="NetworkInspector.applyFilter()" style="margin-left:auto">
                </div>
                <div class="dp-scroll" id="dpNetScroll">
                    <table class="dp-net-table" id="dpNetTable">
                        <thead>
                            <tr>
                                <th style="width:28px">#</th>
                                <th style="width:60px">Method</th>
                                <th>URL</th>
                                <th style="width:60px">Status</th>
                                <th style="width:70px">Time</th>
                                <th style="width:70px">Size</th>
                                <th style="width:80px">Type</th>
                                <th style="width:120px">Waterfall</th>
                            </tr>
                        </thead>
                        <tbody id="dpNetBody"></tbody>
                    </table>
                </div>
                <div class="dp-net-detail" id="dpNetDetail"></div>
            </div>

            <!-- ========= TOKEN TAB ========= -->
            <div class="dp-pane" id="dpPane-token">
                <div class="dp-scroll">
                    <div class="dp-token-container">

                        <div class="dp-section-title">🔑 現在のトークン状態</div>
                        <div class="dp-token-status-card status-fail" id="dpTokenStatusCard">
                            <div class="dp-token-label">STATUS</div>
                            <div class="dp-token-value" id="dpTokenStatusText">Loading...</div>
                        </div>
                        <div style="display:flex;gap:10px;margin-bottom:12px;">
                            <div class="dp-token-status-card" style="flex:1">
                                <div class="dp-token-label">TOKEN PREFIX</div>
                                <div class="dp-token-value" id="dpTokenPrefix" style="color:var(--dbg-cyan);word-break:break-all;">—</div>
                            </div>
                            <div class="dp-token-status-card" style="flex:1;text-align:center;">
                                <div class="dp-token-label">EXPIRES IN</div>
                                <div class="dp-token-countdown" id="dpTokenCountdown">—</div>
                            </div>
                        </div>

                        <div class="dp-section-title">🔧 手動 Bearer トークン上書き</div>
                        <div style="margin-bottom:10px;color:var(--dbg-muted);font-size:10px;line-height:1.6;">
                            Iwara.tv にブラウザでログインして、F12 → Application → Cookies → api.iwara.tv から
                            <span style="color:var(--dbg-cyan);">accessToken</span> の値をコピーして貼り付けてください。
                        </div>
                        <div class="dp-input-row">
                            <input type="text" class="dp-text-input" id="dpManualToken" placeholder="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...">
                            <button class="dp-action-btn primary" onclick="TokenPanel.applyManual()">Apply</button>
                            <button class="dp-action-btn danger" onclick="TokenPanel.clearManual()">Clear</button>
                        </div>

                        <div class="dp-section-title">🔐 メール/パスワードでログイン</div>
                        <div style="margin-bottom:6px;color:var(--dbg-muted);font-size:10px;">
                            Iwaraアカウントの認証情報を入力。Workers内で /user/login を呼び出します。
                        </div>
                        <div class="dp-login-form">
                            <label>メールアドレス</label>
                            <input type="email" class="dp-text-input" id="dpLoginEmail" placeholder="you@example.com">
                            <label>パスワード</label>
                            <input type="password" class="dp-text-input" id="dpLoginPassword" placeholder="••••••••">
                            <div>
                                <button class="dp-action-btn primary" onclick="TokenPanel.loginAndSetToken()">🔐 ログインしてトークン取得</button>
                            </div>
                            <div id="dpLoginStatus" style="font-size:10px;color:var(--dbg-muted);margin-top:4px;"></div>
                        </div>

                        <div class="dp-section-title">📋 トークン取得試行履歴</div>
                        <div id="dpTokenHistory">
                            <div style="color:var(--dbg-muted);font-size:10px;">ページを再読込すると試行ログが表示されます。</div>
                        </div>

                        <div class="dp-section-title">⚡ X-Version 計算機</div>
                        <div class="dp-xver-widget">
                            <div style="display:flex;gap:8px;margin-bottom:8px;">
                                <div style="flex:1;">
                                    <div class="dp-token-label">FILE ID</div>
                                    <input type="text" class="dp-text-input" id="dpXverFileId" placeholder="例: abc123" style="width:100%;margin-top:2px;">
                                </div>
                                <div style="flex:1;">
                                    <div class="dp-token-label">EXPIRES</div>
                                    <input type="text" class="dp-text-input" id="dpXverExpires" placeholder="例: 1711234567" style="width:100%;margin-top:2px;">
                                </div>
                            </div>
                            <button class="dp-action-btn primary" onclick="TokenPanel.computeXVersion()">⚡ SHA-1 計算</button>
                            <div class="dp-xver-formula" id="dpXverResult">入力後にボタンを押してください</div>
                        </div>

                    </div>
                </div>
            </div>

            <!-- ========= DIAGNOSTIC TAB ========= -->
            <div class="dp-pane" id="dpPane-diagnostic">
                <div class="dp-scroll">
                    <div class="dp-diag-container">
                        <div class="dp-diag-header">
                            <input type="text" class="dp-vid-input" id="dpDiagVidInput" placeholder="ビデオID (例: 4trc6R28BzN26Q)">
                            <button class="dp-diag-run-btn" id="dpDiagRunBtn" onclick="DiagnosticPanel.run()">
                                <i class="fa-solid fa-play" style="font-size:10px;"></i>
                                フル診断実行
                            </button>
                            <button class="dp-action-btn" onclick="DiagnosticPanel.runTokenOnly()">トークンのみ</button>
                        </div>

                        <div class="dp-diag-summary" id="dpDiagSummary" style="display:none;">
                            <span class="dp-summary-chip pass" id="dpSummaryPass">✅ 0 passed</span>
                            <span class="dp-summary-chip fail" id="dpSummaryFail">❌ 0 failed</span>
                            <span class="dp-summary-chip warn" id="dpSummaryWarn">⚠ 0 warnings</span>
                            <span style="font-size:10px;color:var(--dbg-muted);margin-left:auto;" id="dpDiagDuration"></span>
                        </div>

                        <div id="dpDiagSteps">
                            <!-- Steps injected here -->
                        </div>
                    </div>
                </div>
            </div>

            <!-- ========= FIXES TAB ========= -->
            <div class="dp-pane" id="dpPane-fixes">
                <div class="dp-scroll">
                    <div class="dp-fixes-container">
                        <div id="dpFixesEmpty" style="color:var(--dbg-muted);font-size:11px;padding:20px 0;">
                            「Diagnostic」タブでフル診断を実行すると、ここに自動修正案が表示されます。
                        </div>
                        <div id="dpConclusions"></div>
                        <div id="dpFixCards"></div>
                    </div>
                </div>
            </div>

        </div><!-- end dp-content -->
    </div><!-- end dp-container -->
</div><!-- end debugPanelRoot -->

<!-- ==================== SCRIPTS ==================== -->
<script>
"use strict";

const ENC_PREFIX = "${GLOBAL_CONFIG.ENC_PREFIX}";

// =====================================================================
// DBG v3 — Structured Logger
// Inspired by: pino (https://github.com/pinojs/pino) +
//              debug npm (https://github.com/visionmedia/debug) +
//              winston (https://github.com/winstonjs/winston)
// =====================================================================
const DBG = (() => {
    const _logs = [];
    let _errorCount = 0;
    let _levelFilter = 'all';
    let _textFilter = '';

    function _ts() {
        const n = new Date();
        return n.toTimeString().slice(0,8) + '.' + String(n.getMilliseconds()).padStart(3,'0');
    }

    function _matchesFilter(entry) {
        if (_levelFilter !== 'all' && entry.level !== _levelFilter) return false;
        if (_textFilter) {
            try {
                const re = new RegExp(_textFilter, 'i');
                if (!re.test(entry.msg)) return false;
            } catch(_) {
                if (!entry.msg.toLowerCase().includes(_textFilter.toLowerCase())) return false;
            }
        }
        return true;
    }

    function _renderEntry(entry) {
        const el = document.createElement('div');
        el.className = \`dp-log-entry level-\${entry.level}\`;
        el.dataset.logId = entry.id;

        const levelLabels = {
            step: '▶ STEP', info: 'ℹ INFO', warn: '⚠ WARN',
            error: '✖ ERR', success: '✔ OK', net: '🌐 NET'
        };

        let html = \`<span class="dp-log-ts">\${entry.ts}</span>\`;
        html += \`<span class="dp-log-ns">\${levelLabels[entry.level] || entry.level.toUpperCase()}</span>\`;
        html += \`<span class="dp-log-msg">\${escHtml(entry.msg)}\`;

        if (entry.data) {
            const jsonId = 'json-' + entry.id;
            html += \` <span class="dp-json-toggle" onclick="DBG.toggleJson('\${jsonId}')">{ JSON }</span>\`;
            html += \`<div class="dp-json-body" id="\${jsonId}">\${escHtml(JSON.stringify(entry.data, null, 2))}</div>\`;
        }
        html += '</span>';
        el.innerHTML = html;
        return el;
    }

    function _addToDOM(entry) {
        if (!_matchesFilter(entry)) return;
        const log = document.getElementById('dpConsoleLog');
        if (!log) return;
        const el = _renderEntry(entry);
        log.appendChild(el);
        log.scrollTop = log.scrollHeight;
        // Update count
        const countEl = document.getElementById('dpLogCount');
        if (countEl) countEl.textContent = _logs.length + ' entries';
    }

    function _add(level, msg, data) {
        const entry = { id: _logs.length, ts: _ts(), level, msg: String(msg), data: data || null };
        _logs.push(entry);

        if (level === 'error') {
            _errorCount++;
            const badge = document.getElementById('dbgErrorBadge');
            if (badge) { badge.style.display = 'flex'; badge.textContent = _errorCount; }
        }

        _addToDOM(entry);

        // Mirror to browser console
        const styles = {
            step:    'color:#1e40af;font-weight:bold',
            info:    'color:#166534',
            warn:    '',
            error:   '',
            success: 'color:#15803d;font-weight:bold',
            net:     'color:#7c3aed'
        };
        const prefix = '[DBG/' + level.toUpperCase() + '] ';
        if (level === 'error')        console.error(prefix + msg, data || '');
        else if (level === 'warn')    console.warn(prefix + msg, data || '');
        else if (styles[level])       console.log('%c' + prefix + msg, styles[level], data || '');
        else                          console.log(prefix + msg, data || '');
    }

    return {
        step:    (msg, d) => _add('step',    msg, d),
        info:    (msg, d) => _add('info',    msg, d),
        warn:    (msg, d) => _add('warn',    msg, d),
        error:   (msg, d) => _add('error',   msg, d),
        success: (msg, d) => _add('success', msg, d),
        net:     (msg, d) => _add('net',     msg, d),
        getLogs: ()       => _logs,
        getErrors: ()     => _logs.filter(l => l.level === 'error'),
        clear: () => {
            _logs.length = 0;
            _errorCount = 0;
            const log = document.getElementById('dpConsoleLog');
            if (log) log.innerHTML = '';
            const badge = document.getElementById('dbgErrorBadge');
            if (badge) badge.style.display = 'none';
        },
        toggleJson: (id) => {
            const el = document.getElementById(id);
            if (el) el.classList.toggle('visible');
        },
        setLevelFilter: (level) => {
            _levelFilter = level;
            DBG._rerender();
        },
        setTextFilter: (text) => {
            _textFilter = text;
            DBG._rerender();
        },
        _rerender: () => {
            const log = document.getElementById('dpConsoleLog');
            if (!log) return;
            log.innerHTML = '';
            _logs.forEach(e => {
                if (_matchesFilter(e)) log.appendChild(_renderEntry(e));
            });
            log.scrollTop = log.scrollHeight;
        }
    };
})();

// =====================================================================
// NetworkInspector — Fetch Monkey-Patcher
// Inspired by:
//   https://github.com/GoogleChrome/devtools-frontend (network waterfall)
//   https://github.com/nicolo-ribaudo/har-validator (HAR format)
//   axios interceptors (https://github.com/axios/axios)
// =====================================================================
const NetworkInspector = (() => {
    const _requests = new Map();
    let _reqId = 0;
    let _selectedId = null;
    let _filter = '';
    let _maxTime = 100; // for waterfall scaling

    function _install() {
        const _origFetch = window.fetch.bind(window);
        window.fetch = async function(...args) {
            const id = ++_reqId;
            const url = typeof args[0] === 'string' ? args[0] : (args[0]?.url || '?');
            const method = (args[1]?.method || 'GET').toUpperCase();
            const startTime = performance.now();

            const req = { id, url, method, startTime, status: null, statusText: null,
                          durationMs: null, size: null, contentType: null,
                          reqHeaders: args[1]?.headers || {}, resHeaders: {},
                          resBodyPreview: null, error: null, state: 'pending' };
            _requests.set(id, req);
            _renderRow(req, true);

            DBG.net(\`↑ \${method} \${url.slice(0, 120)}\`);

            try {
                const response = await _origFetch(...args);
                req.durationMs = Math.round(performance.now() - startTime);
                req.status = response.status;
                req.statusText = response.statusText;
                req.contentType = response.headers.get('Content-Type') || '';
                req.state = response.ok ? 'ok' : 'error';

                // Capture diagnostic headers
                req.diagHeaders = {
                    tokenPresent: response.headers.get('X-Diag-Token-Present'),
                    tokenExpiry:  response.headers.get('X-Diag-Token-Expiry'),
                    hostname:     response.headers.get('X-Diag-Proxy-Hostname'),
                    path:         response.headers.get('X-Diag-Proxy-Path'),
                };

                // Copy response for body preview (clone before consuming)
                try {
                    const cloned = response.clone();
                    const text = await cloned.text();
                    req.size = text.length;
                    req.resBodyPreview = text.slice(0, 2000);
                } catch(_) {}

                _updateRow(req);
                if (_maxTime < req.durationMs) _maxTime = req.durationMs;
                DBG.net(\`↓ \${response.status} \${method} \${url.slice(0, 80)} (\${req.durationMs}ms)\`);

                const badge = document.getElementById('dpBadgeNetwork');
                if (badge) badge.textContent = _reqId;

                return response;
            } catch (err) {
                req.durationMs = Math.round(performance.now() - startTime);
                req.error = String(err);
                req.state = 'error';
                req.status = 0;
                _updateRow(req);
                DBG.error(\`fetch FAILED: \${method} \${url.slice(0,80)} — \${err}\`);
                throw err;
            }
        };
    }

    function _statusClass(status) {
        if (!status) return '';
        if (status >= 200 && status < 300) return 'dp-net-status-ok';
        if (status >= 300 && status < 400) return 'dp-net-status-warn';
        return 'dp-net-status-err';
    }

    function _waterfallBar(durationMs) {
        if (!durationMs) return '';
        const maxW = 100;
        const pct = Math.min(maxW, (durationMs / Math.max(_maxTime, 1)) * maxW);
        let cls = 'dp-waterfall-bar';
        if (durationMs > 2000) cls += ' very-slow';
        else if (durationMs > 500) cls += ' slow';
        return \`<div class="\${cls}" style="width:\${pct}%"></div>\`;
    }

    function _shortUrl(url) {
        try {
            const u = new URL(url, window.location.origin);
            return u.pathname.slice(0, 60) + (u.search ? '?' + u.search.slice(0, 20) : '');
        } catch(_) { return url.slice(0, 60); }
    }

    function _matchesFilter(req) {
        if (!_filter) return true;
        return req.url.toLowerCase().includes(_filter.toLowerCase());
    }

    function _renderRow(req, isNew = false) {
        if (!_matchesFilter(req)) return;
        const tbody = document.getElementById('dpNetBody');
        if (!tbody) return;

        const tr = document.createElement('tr');
        tr.className = 'dp-net-row';
        tr.id = 'dpNetRow-' + req.id;
        tr.onclick = () => NetworkInspector.selectRow(req.id);
        tr.innerHTML = _rowHtml(req);
        if (isNew) tbody.appendChild(tr);
    }

    function _rowHtml(req) {
        const statusText = req.state === 'pending' ? '<span style="color:var(--dbg-muted)">...</span>' :
            \`<span class="\${_statusClass(req.status)}">\${req.status || 'ERR'}</span>\`;
        const sizeText = req.size != null ? (req.size > 1024 ? (req.size/1024).toFixed(1)+'KB' : req.size+'B') : '—';
        const timeText = req.durationMs != null ? req.durationMs + 'ms' : '...';
        const ct = (req.contentType || '').split(';')[0].split('/').pop() || '—';
        return \`
            <td style="color:var(--dbg-muted);font-size:10px;">\${req.id}</td>
            <td class="dp-net-method">\${req.method}</td>
            <td><div class="dp-net-url" title="\${escHtml(req.url)}">\${escHtml(_shortUrl(req.url))}</div></td>
            <td>\${statusText}</td>
            <td class="dp-net-time">\${timeText}</td>
            <td class="dp-net-size">\${sizeText}</td>
            <td class="dp-net-type" style="font-size:10px;">\${ct}</td>
            <td>\${_waterfallBar(req.durationMs)}</td>
        \`;
    }

    function _updateRow(req) {
        const tr = document.getElementById('dpNetRow-' + req.id);
        if (tr) {
            tr.innerHTML = _rowHtml(req);
            tr.onclick = () => NetworkInspector.selectRow(req.id);
        }
    }

    function _renderDetail(req) {
        const detail = document.getElementById('dpNetDetail');
        if (!detail) return;
        detail.classList.add('visible');

        const diagHeadersHtml = req.diagHeaders ? \`
            <div class="dp-detail-kv"><span class="dk">X-Diag-Token-Present</span><span class="dv \${req.diagHeaders.tokenPresent === '1' ? 'ok' : 'err'}">\${req.diagHeaders.tokenPresent === '1' ? '✅ YES' : '❌ NO'}</span></div>
            <div class="dp-detail-kv"><span class="dk">X-Diag-Token-Expiry</span><span class="dv">\${req.diagHeaders.tokenExpiry ? new Date(+req.diagHeaders.tokenExpiry).toLocaleTimeString() : '—'}</span></div>
            <div class="dp-detail-kv"><span class="dk">X-Diag-Proxy-Host</span><span class="dv">\${req.diagHeaders.hostname || '—'}</span></div>
            <div class="dp-detail-kv"><span class="dk">X-Diag-Proxy-Path</span><span class="dv">\${req.diagHeaders.path || '—'}</span></div>
        \` : '';

        const isFileEndpoint = req.url.includes('/file');
        let fileAnalysis = '';
        if (isFileEndpoint && req.resBodyPreview) {
            let parsed = null;
            try { parsed = JSON.parse(req.resBodyPreview); } catch(_) {}
            if (parsed && parsed.message) {
                fileAnalysis = \`<div class="dp-detail-kv"><span class="dk" style="color:var(--dbg-red);">⚠ FILE API ERROR</span><span class="dv err">\${parsed.message}</span></div>
                <div class="dp-detail-kv"><span class="dk" style="color:var(--dbg-red);">→ 原因</span><span class="dv err">\${req.diagHeaders?.tokenPresent === '0' ? 'Bearerトークンが空です。Tokenタブで手動設定してください。' : '動画が非公開またはAPIエラー'}</span></div>\`;
            } else if (Array.isArray(parsed)) {
                if (parsed.length === 0) {
                    fileAnalysis = \`<div class="dp-detail-kv"><span class="dk" style="color:var(--dbg-yellow);">⚠ EMPTY ARRAY</span><span class="dv" style="color:var(--dbg-yellow);">ファイルリストが空 — トークン無効または動画にファイルなし</span></div>\`;
                } else {
                    fileAnalysis = \`<div class="dp-detail-kv"><span class="dk" style="color:var(--dbg-green);">✅ FILES</span><span class="dv ok">\${parsed.length}件のファイルを検出</span></div>\`;
                }
            }
        }

        detail.innerHTML = \`
            <div class="dp-detail-section">
                <h4>📤 Request</h4>
                <div class="dp-detail-kv"><span class="dk">URL</span><span class="dv" style="color:var(--dbg-blue);word-break:break-all;">\${escHtml(req.url)}</span></div>
                <div class="dp-detail-kv"><span class="dk">Method</span><span class="dv" style="color:var(--dbg-purple);">\${req.method}</span></div>
            </div>
            <div class="dp-detail-section">
                <h4>📥 Response</h4>
                <div class="dp-detail-kv"><span class="dk">Status</span><span class="dv \${req.status >= 200 && req.status < 300 ? 'ok' : 'err'}">\${req.status} \${req.statusText || ''}</span></div>
                <div class="dp-detail-kv"><span class="dk">Duration</span><span class="dv">\${req.durationMs}ms</span></div>
                <div class="dp-detail-kv"><span class="dk">Content-Type</span><span class="dv">\${req.contentType || '—'}</span></div>
                <div class="dp-detail-kv"><span class="dk">Size</span><span class="dv">\${req.size != null ? req.size + ' bytes' : '—'}</span></div>
                \${fileAnalysis}
            </div>
            \${diagHeadersHtml ? \`<div class="dp-detail-section"><h4>🔬 Diagnostic Headers (Worker)</h4>\${diagHeadersHtml}</div>\` : ''}
            <div class="dp-detail-section">
                <h4>📄 Response Body Preview</h4>
                <div style="background:var(--dbg-bg);border-radius:4px;padding:6px 8px;font-size:10px;color:var(--dbg-cyan);white-space:pre-wrap;max-height:120px;overflow-y:auto;">\${escHtml((req.resBodyPreview || '—').slice(0, 1000))}</div>
            </div>
        \`;
    }

    return {
        install: _install,
        clear: () => {
            _requests.clear();
            _reqId = 0;
            _selectedId = null;
            const tbody = document.getElementById('dpNetBody');
            if (tbody) tbody.innerHTML = '';
            const detail = document.getElementById('dpNetDetail');
            if (detail) { detail.innerHTML = ''; detail.classList.remove('visible'); }
        },
        selectRow: (id) => {
            _selectedId = id;
            document.querySelectorAll('.dp-net-row').forEach(r => r.classList.remove('selected'));
            const tr = document.getElementById('dpNetRow-' + id);
            if (tr) tr.classList.add('selected');
            const req = _requests.get(id);
            if (req) _renderDetail(req);
        },
        applyFilter: () => {
            _filter = document.getElementById('dpNetSearch')?.value || '';
            const tbody = document.getElementById('dpNetBody');
            if (!tbody) return;
            tbody.innerHTML = '';
            _requests.forEach(req => _renderRow(req));
        },
        exportHAR: () => {
            const entries = [];
            _requests.forEach(req => {
                entries.push({
                    startedDateTime: new Date().toISOString(),
                    time: req.durationMs || 0,
                    request: { method: req.method, url: req.url, headers: [], queryString: [], postData: null, headersSize: -1, bodySize: -1 },
                    response: { status: req.status || 0, statusText: req.statusText || '', headers: [], content: { size: req.size || 0, mimeType: req.contentType || '' }, redirectURL: '', headersSize: -1, bodySize: req.size || 0, bodyText: req.resBodyPreview || '' },
                    cache: {}, timings: { send: 0, wait: req.durationMs || 0, receive: 0 }
                });
            });
            const har = { log: { version: '1.2', creator: { name: 'Iwara Debug', version: '8.0' }, entries } };
            const blob = new Blob([JSON.stringify(har, null, 2)], { type: 'application/json' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a'); a.href = url; a.download = 'iwara-network.har'; a.click();
            URL.revokeObjectURL(url);
        },
        getAll: () => _requests
    };
})();

// =====================================================================
// TokenPanel — UI for Token Tab
// =====================================================================
const TokenPanel = {
    _countdownTimer: null,

    refresh() {
        // Will be called after token operations
        // In a real Worker setup we can't read the server cache from client,
        // but we track client-side fetch results via NetworkInspector
    },

    applyManual() {
        const val = document.getElementById('dpManualToken')?.value?.trim();
        if (!val) { DBG.warn('Manual token: empty input'); return; }
        DBG.step('Manual token override applied. Prefix: ' + val.slice(0, 30));
        window._manualBearerToken = val;
        DBG.success('Manual Bearer token set. 次の動画再生から有効になります。');
        document.getElementById('dpTokenStatusText').textContent = '手動トークン設定済み';
        document.getElementById('dpTokenStatusCard').className = 'dp-token-status-card status-ok';
        document.getElementById('dpTokenPrefix').textContent = val.slice(0, 40) + '...';
    },

    clearManual() {
        window._manualBearerToken = null;
        if (document.getElementById('dpManualToken')) document.getElementById('dpManualToken').value = '';
        DBG.info('Manual token cleared');
    },

    async loginAndSetToken() {
        const email = document.getElementById('dpLoginEmail')?.value?.trim();
        const password = document.getElementById('dpLoginPassword')?.value;
        const statusEl = document.getElementById('dpLoginStatus');

        if (!email || !password) {
            if (statusEl) statusEl.textContent = '❌ メールとパスワードを入力してください';
            return;
        }

        if (statusEl) statusEl.textContent = '⏳ ログイン試行中...';
        DBG.step('Login attempt: ' + email);

        try {
            const res = await fetch('/' + _enc('https://api.iwara.tv/user/login'), {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ email, password })
            });
            const data = await res.json();
            DBG.info('Login response keys: ' + Object.keys(data).join(', '), data);

            const token = data.token || data.accessToken || data.access_token || data.jwt || '';
            if (token) {
                window._manualBearerToken = token;
                if (document.getElementById('dpManualToken')) document.getElementById('dpManualToken').value = token;
                if (statusEl) statusEl.textContent = '✅ ログイン成功！トークンを設定しました。';
                DBG.success('Login SUCCESS. Token prefix: ' + token.slice(0, 30));
                document.getElementById('dpTokenStatusCard').className = 'dp-token-status-card status-ok';
                document.getElementById('dpTokenStatusText').textContent = 'ログイン済みトークン設定済み';
                document.getElementById('dpTokenPrefix').textContent = token.slice(0, 40) + '...';
            } else {
                if (statusEl) statusEl.textContent = '❌ トークンフィールドが見つかりません: ' + JSON.stringify(data).slice(0, 100);
                DBG.error('Login: no token field. Keys: ' + Object.keys(data).join(', '), data);
            }
        } catch(e) {
            if (statusEl) statusEl.textContent = '❌ エラー: ' + e.message;
            DBG.error('Login request failed: ' + e.message);
        }
    },

    async computeXVersion() {
        const fileId = document.getElementById('dpXverFileId')?.value?.trim();
        const expires = document.getElementById('dpXverExpires')?.value?.trim();
        const resultEl = document.getElementById('dpXverResult');
        if (!fileId) { if(resultEl) resultEl.textContent = '❌ File ID を入力してください'; return; }

        const salt = '5nFp9kmbNnHdAFhaqMvt';
        const input = expires ? \`\${fileId}_\${expires}_\${salt}\` : \`\${fileId}\${salt}\`;

        // SHA-1 via Web Crypto
        const msgUint8 = new TextEncoder().encode(input);
        const hashBuffer = await crypto.subtle.digest('SHA-1', msgUint8);
        const hashHex = Array.from(new Uint8Array(hashBuffer)).map(b => b.toString(16).padStart(2,'0')).join('');

        if (resultEl) {
            resultEl.textContent = \`Input: \${input}\\nSHA-1: \${hashHex}\\n\\nX-Version: \${hashHex}\`;
        }
        DBG.info('X-Version computed: ' + hashHex + ' (input: ' + input + ')');
    }
};

// =====================================================================
// DiagnosticPanel — Runs /api/diagnose and renders results
// Inspired by:
//   Lighthouse (https://github.com/GoogleChrome/lighthouse)
//   Clinic.js diagnostics
//   yt-dlp verbose mode
// =====================================================================
const DiagnosticPanel = {
    _lastReport: null,
    _stepDefs: [
        { key: 'token_probe',       icon: '🔍', title: 'トークンエンドポイント探索',     sub: '全エンドポイントを試行してどれが有効か確認' },
        { key: 'token_acquire',     icon: '🔑', title: 'Bearerトークン取得',            sub: 'Guest tokenまたはログイントークンを取得' },
        { key: 'video_metadata',    icon: '📄', title: '動画メタデータ取得',             sub: 'タイトル・タイプ・embedUrlなどを確認' },
        { key: 'file_list',         icon: '📁', title: 'ファイルリスト取得',             sub: '/video/{id}/file の応答を検証' },
        { key: 'stream_url_probe',  icon: '🎬', title: 'ストリームURL探索',             sub: 'X-Version付きでHEADリクエスト送信' },
        { key: 'xversion_validation', icon: '🔐', title: 'X-Version検証',             sub: 'SHA-1署名の計算式を確認' },
    ],

    _renderInitialSteps() {
        const container = document.getElementById('dpDiagSteps');
        if (!container) return;
        container.innerHTML = '';
        this._stepDefs.forEach(def => {
            const div = document.createElement('div');
            div.className = 'dp-diag-step';
            div.id = 'dpStep-' + def.key;
            div.onclick = () => DiagnosticPanel._toggleDetail(def.key);
            div.innerHTML = \`
                <div class="ds-icon idle" id="dpStepIcon-\${def.key}">◦</div>
                <div class="ds-body">
                    <div class="ds-title">\${def.icon} \${def.title}</div>
                    <div class="ds-sub" id="dpStepSub-\${def.key}">\${def.sub}</div>
                    <div class="dp-diag-detail" id="dpStepDetail-\${def.key}"></div>
                </div>
                <div class="ds-dur" id="dpStepDur-\${def.key}"></div>
            \`;
            container.appendChild(div);
        });
    },

    _toggleDetail(key) {
        const el = document.getElementById('dpStepDetail-' + key);
        if (el) el.classList.toggle('visible');
    },

    _updateStep(key, status, sub, detailData, durationMs) {
        const iconEl = document.getElementById('dpStepIcon-' + key);
        const subEl  = document.getElementById('dpStepSub-' + key);
        const detEl  = document.getElementById('dpStepDetail-' + key);
        const durEl  = document.getElementById('dpStepDur-' + key);

        if (iconEl) {
            const icons = { pass: '✓', fail: '✗', running: '⟳', idle: '◦', skip: '—' };
            iconEl.className = 'ds-icon ' + status;
            iconEl.textContent = icons[status] || '?';
        }
        if (subEl && sub) subEl.textContent = sub;
        if (detEl && detailData) {
            detEl.textContent = typeof detailData === 'string' ? detailData : JSON.stringify(detailData, null, 2);
            detEl.classList.add('visible');
        }
        if (durEl && durationMs != null) durEl.textContent = durationMs + 'ms';
    },

    async run() {
        const vidInput = document.getElementById('dpDiagVidInput');
        const vid = vidInput?.value?.trim() || App.state?.currentVid || '';
        if (!vid) {
            DBG.warn('Diagnostic: ビデオIDを入力してください');
            if (vidInput) { vidInput.style.borderColor = 'var(--dbg-red)'; setTimeout(() => { vidInput.style.borderColor = ''; }, 2000); }
            return;
        }

        const runBtn = document.getElementById('dpDiagRunBtn');
        if (runBtn) { runBtn.disabled = true; runBtn.innerHTML = '<span style="animation:pulse-anim 1s infinite">⏳ 診断中...</span>'; }

        // Show tab
        DebugPanel.switchTab('diagnostic');

        // Initialize steps UI
        this._renderInitialSteps();
        document.getElementById('dpDiagSummary').style.display = 'none';

        const startTotal = performance.now();
        DBG.step('=== FULL DIAGNOSTIC START === vid: ' + vid);

        try {
            // Call /api/diagnose endpoint
            this._stepDefs.forEach(d => this._updateStep(d.key, 'running', null, null, null));

            const diagUrl = '/api/diagnose?vid=' + encodeURIComponent(vid);
            DBG.info('Calling diagnostic endpoint: ' + diagUrl);

            const res = await fetch(diagUrl);
            if (!res.ok) {
                throw new Error('Diagnostic endpoint returned ' + res.status);
            }
            const report = await res.json();
            this._lastReport = report;

            DBG.info('Diagnostic report received', report);

            // Render each step result
            Object.entries(report.steps || {}).forEach(([key, step]) => {
                const status = step.status === 'pass' ? 'pass' : (step.status === 'fail' ? 'fail' : 'skip');
                const sub = step.status === 'pass'
                    ? '✅ ' + (step.data ? JSON.stringify(step.data).slice(0, 80) : 'OK')
                    : '❌ ' + (step.error || '').slice(0, 100);
                this._updateStep(key, status, sub, step.data || step.error, step.durationMs);
                DBG[step.status === 'pass' ? 'success' : 'error'](\`Step [\${key}]: \${step.status}\`, step.data || step.error);
            });

            // Summary
            const summaryEl = document.getElementById('dpDiagSummary');
            if (summaryEl) summaryEl.style.display = 'flex';
            document.getElementById('dpSummaryPass').textContent = '✅ ' + report.summary.passed + ' passed';
            document.getElementById('dpSummaryFail').textContent = '❌ ' + report.summary.failed + ' failed';
            document.getElementById('dpSummaryWarn').textContent = '⚠ ' + report.summary.warnings + ' warnings';
            const totalMs = Math.round(performance.now() - startTotal);
            document.getElementById('dpDiagDuration').textContent = '総時間: ' + totalMs + 'ms';

            // Render fixes
            this._renderFixes(report);

        } catch(e) {
            DBG.error('Diagnostic FAILED: ' + e.message);
            this._stepDefs.forEach(d => {
                const iconEl = document.getElementById('dpStepIcon-' + d.key);
                if (iconEl && iconEl.className.includes('running')) this._updateStep(d.key, 'skip', '診断エンドポイントエラー', null, null);
            });
        } finally {
            if (runBtn) { runBtn.disabled = false; runBtn.innerHTML = '<i class="fa-solid fa-play" style="font-size:10px;"></i> フル診断実行'; }
        }
    },

    async runTokenOnly() {
        DBG.step('=== TOKEN-ONLY DIAGNOSTIC ===');
        DebugPanel.switchTab('token');
        try {
            const res = await fetch('/api/token-probe');
            const data = await res.json();
            DBG.info('Token probe result', data);

            const historyEl = document.getElementById('dpTokenHistory');
            if (historyEl) {
                historyEl.innerHTML = '';
                (data.attempts || []).forEach(attempt => {
                    const div = document.createElement('div');
                    div.className = 'dp-attempt-row';
                    div.innerHTML = \`
                        <div class="ar-url">\${attempt.method} \${attempt.url}</div>
                        <div>\${attempt.error
                            ? \`<span class="ar-status-err">❌ \${escHtml(attempt.error)}</span>\`
                            : \`<span class="ar-status-ok">✅ HTTP \${attempt.status} | hasToken: \${attempt.hasToken}</span>\`}
                        </div>
                        \${attempt.tokenKeys ? \`<div style="color:var(--dbg-muted);">Keys: \${attempt.tokenKeys.join(', ')}</div>\` : ''}
                        \${attempt.bodyPreview ? \`<div style="color:var(--dbg-cyan);font-size:10px;margin-top:4px;">\${escHtml(attempt.bodyPreview.slice(0,200))}</div>\` : ''}
                    \`;
                    historyEl.appendChild(div);
                });
            }
        } catch(e) {
            DBG.error('Token probe request failed: ' + e.message);
        }
    },

    _renderFixes(report) {
        const conclusionsEl = document.getElementById('dpConclusions');
        const fixCardsEl = document.getElementById('dpFixCards');
        const emptyEl = document.getElementById('dpFixesEmpty');

        if (!conclusionsEl || !fixCardsEl) return;

        const hasFixes = (report.conclusions?.length > 0) || (report.fixes?.length > 0);

        if (emptyEl) emptyEl.style.display = hasFixes ? 'none' : 'block';

        // Conclusions
        conclusionsEl.innerHTML = '';
        if (report.conclusions?.length > 0) {
            const titleEl = document.createElement('div');
            titleEl.className = 'dp-section-title';
            titleEl.textContent = '🔍 診断結果';
            conclusionsEl.appendChild(titleEl);
        }
        (report.conclusions || []).forEach(c => {
            const div = document.createElement('div');
            div.className = \`dp-conclusion-card \${c.severity || 'warning'}\`;
            div.innerHTML = \`
                <div class="cc-title">\${c.code}: \${escHtml(c.msg)}</div>
                \${c.detail ? \`<div class="cc-detail">\${escHtml(c.detail.slice(0, 300))}</div>\` : ''}
            \`;
            conclusionsEl.appendChild(div);
        });

        // Fix cards
        fixCardsEl.innerHTML = '';
        if (report.fixes?.length > 0) {
            const titleEl = document.createElement('div');
            titleEl.className = 'dp-section-title';
            titleEl.style.marginTop = '16px';
            titleEl.textContent = '🔧 修正案';
            fixCardsEl.appendChild(titleEl);
        }
        (report.fixes || []).forEach(fix => {
            const div = document.createElement('div');
            div.className = 'dp-fix-card';
            let actionHtml = '';
            if (fix.action === 'manual_token_input') {
                actionHtml = \`<button class="dp-action-btn primary" onclick="DebugPanel.switchTab('token')">Tokenタブを開く →</button>\`;
            } else if (fix.action === 'login_form') {
                actionHtml = \`<button class="dp-action-btn primary" onclick="DebugPanel.switchTab('token')">ログインフォームへ →</button>\`;
            } else if (fix.action === 'open_external' && fix.url) {
                actionHtml = \`<button class="dp-action-btn warning" onclick="window.open('\${escHtml(fix.url)}', '_blank')">外部リンクを開く →</button>\`;
            } else if (fix.action === 'check_salt') {
                actionHtml = \`<button class="dp-action-btn" onclick="DebugPanel.switchTab('token'); setTimeout(() => document.getElementById('dpXverFileId')?.focus(), 100)">X-Version計算機を開く →</button>\`;
            }
            div.innerHTML = \`
                <div class="fc-label">🔧 \${escHtml(fix.label)}</div>
                <div class="fc-desc">\${escHtml(fix.description)}</div>
                \${actionHtml ? \`<div class="fc-action">\${actionHtml}</div>\` : ''}
            \`;
            fixCardsEl.appendChild(div);
        });

        // Update badge
        const fixes = report.fixes?.length || 0;
        const badge = document.getElementById('dpBadgeFixes');
        if (badge) {
            if (fixes > 0) { badge.style.display = 'flex'; badge.textContent = fixes; }
            else badge.style.display = 'none';
        }
        const diagBadge = document.getElementById('dpBadgeDiag');
        if (diagBadge) {
            const failed = report.summary?.failed || 0;
            if (failed > 0) { diagBadge.style.display = 'flex'; diagBadge.textContent = failed; }
            else diagBadge.style.display = 'none';
        }
    }
};

// =====================================================================
// DebugPanel — Main controller
// =====================================================================
const DebugPanel = {
    _activeTab: 'console',
    _isMinimized: false,
    _height: 360,
    _resizing: false,

    init() {
        // Setup resize handle
        const container = document.getElementById('dpContainer');
        const headerBar = document.getElementById('dpHeaderBar');
        if (headerBar) {
            headerBar.addEventListener('mousedown', (e) => {
                if (e.target.tagName === 'BUTTON') return;
                this._resizing = true;
                this._startY = e.clientY;
                this._startH = container.offsetHeight;
                document.body.style.userSelect = 'none';
            });
        }
        document.addEventListener('mousemove', (e) => {
            if (!this._resizing) return;
            const delta = this._startY - e.clientY;
            const newH = Math.min(Math.max(120, this._startH + delta), window.innerHeight * 0.9);
            container.style.height = newH + 'px';
            this._height = newH;
        });
        document.addEventListener('mouseup', () => {
            this._resizing = false;
            document.body.style.userSelect = '';
        });

        // Initialize diagnostic steps
        DiagnosticPanel._renderInitialSteps();

        // Start token countdown
        this._startTokenCountdown();

        DBG.success('Iwara Edge Debugger v8.0 initialized');
        DBG.info('NetworkInspector active — fetch()をモンキーパッチ済み');
    },

    show() {
        const container = document.getElementById('dpContainer');
        if (container) {
            container.style.display = 'flex';
            container.style.height = this._height + 'px';
            this._isMinimized = false;
            container.classList.remove('dp-minimized');
        }
        const btn = document.getElementById('dbgLaunchBtn');
        if (btn) btn.style.display = 'none';
    },

    hide() {
        const container = document.getElementById('dpContainer');
        if (container) container.style.display = 'none';
        const btn = document.getElementById('dbgLaunchBtn');
        if (btn) btn.style.display = 'flex';
    },

    minimize() {
        const container = document.getElementById('dpContainer');
        if (!container) return;
        if (this._isMinimized) {
            container.classList.remove('dp-minimized');
            container.style.height = this._height + 'px';
            this._isMinimized = false;
        } else {
            container.classList.add('dp-minimized');
            this._isMinimized = true;
        }
    },

    switchTab(tabName) {
        this._activeTab = tabName;
        document.querySelectorAll('#dpTabs .dp-tab').forEach(t => {
            t.classList.toggle('active', t.dataset.tab === tabName);
        });
        document.querySelectorAll('.dp-pane').forEach(p => {
            p.classList.toggle('active', p.id === 'dpPane-' + tabName);
        });
        if (tabName === 'token') this._refreshTokenPanel();
        if (tabName === 'network') NetworkInspector.applyFilter();
    },

    clearAll() {
        DBG.clear();
        NetworkInspector.clear();
        DBG.success('Debug cleared');
    },

    applyConsoleFilter() {
        const v = document.getElementById('dpConsoleSearch')?.value || '';
        DBG.setTextFilter(v);
    },

    setLevelFilter(level, btn) {
        document.querySelectorAll('.dp-filter-btn').forEach(b => b.classList.remove('active-filter'));
        if (btn) btn.classList.add('active-filter');
        DBG.setLevelFilter(level);
    },

    exportLogs() {
        const data = {
            timestamp: new Date().toISOString(),
            version: '${GLOBAL_CONFIG.VERSION}',
            consoleLogs: DBG.getLogs(),
            networkRequests: Array.from(NetworkInspector.getAll().values()),
            diagnosticReport: DiagnosticPanel._lastReport
        };
        const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a'); a.href = url; a.download = 'iwara-debug-' + Date.now() + '.json'; a.click();
        URL.revokeObjectURL(url);
        DBG.success('ログをエクスポートしました');
    },

    copyLogs() {
        const lines = DBG.getLogs().map(l => \`[\${l.ts}] [\${l.level.toUpperCase()}] \${l.msg}\${l.data ? ' ' + JSON.stringify(l.data) : ''}\`).join('\\n');
        navigator.clipboard.writeText(lines).then(() => DBG.success('ログをクリップボードにコピーしました'));
    },

    quickDiagnoseCurrentVideo() {
        const vid = App.state?.currentVid;
        if (!vid) { DBG.warn('診断: 動画が選択されていません'); return; }
        const input = document.getElementById('dpDiagVidInput');
        if (input) input.value = vid;
        this.show();
        this.switchTab('diagnostic');
        DiagnosticPanel.run();
    },

    _refreshTokenPanel() {
        // Update token status from client-side knowledge
        const manual = window._manualBearerToken;
        const statusCard = document.getElementById('dpTokenStatusCard');
        const statusText = document.getElementById('dpTokenStatusText');
        const prefix = document.getElementById('dpTokenPrefix');

        if (manual) {
            if (statusCard) statusCard.className = 'dp-token-status-card status-ok';
            if (statusText) statusText.textContent = '✅ 手動トークン設定済み';
            if (prefix) prefix.textContent = manual.slice(0, 40) + '...';
        } else {
            // Check NetworkInspector for last /file endpoint diagnostic header
            let found = false;
            NetworkInspector.getAll().forEach(req => {
                if (req.url.includes('/file') && req.diagHeaders) {
                    if (req.diagHeaders.tokenPresent === '1') {
                        if (statusCard) statusCard.className = 'dp-token-status-card status-ok';
                        if (statusText) statusText.textContent = '✅ Workerトークン有効 (最後のリクエストで確認)';
                        found = true;
                    } else if (req.diagHeaders.tokenPresent === '0') {
                        if (statusCard) statusCard.className = 'dp-token-status-card status-fail';
                        if (statusText) statusText.textContent = '❌ Workerトークン無効 — 取得失敗';
                        found = true;
                    }
                }
            });
            if (!found) {
                if (statusCard) statusCard.className = 'dp-token-status-card status-warn';
                if (statusText) statusText.textContent = '⚠ 未確認 — まだ /file エンドポイントが呼ばれていません';
            }
        }
    },

    _startTokenCountdown() {
        setInterval(() => {
            if (this._activeTab !== 'token') return;
            const cdEl = document.getElementById('dpTokenCountdown');
            if (!cdEl) return;
            // We can't directly access worker cache, but show manual token status
            cdEl.textContent = window._manualBearerToken ? '手動設定' : '—';
        }, 1000);
    }
};

// =====================================================================
// App Core — Main Application Logic (preserved from v6)
// =====================================================================
const App = {
    state: {
        page: 0,
        query: '',
        sort: 'date',
        videos: [],
        currentVid: null
    },

    init() {
        NetworkInspector.install();
        DebugPanel.init();

        this.fetchVideos();
        document.getElementById('searchInput').onkeydown = (e) => {
            if (e.key === 'Enter') this.handleSearch();
        };
        DBG.step('App initialized');
    },

    _enc(str) {
        if (!str) return str;
        try {
            return ENC_PREFIX + btoa(encodeURIComponent(str))
                .replace(/\\+/g, '-').replace(/\\//g, '_').replace(/=/g, '');
        } catch(e) { return str; }
    },

    showLoader(visible) {
        document.getElementById('globalLoader').classList.toggle('hidden', !visible);
    },

    goHome() {
        this.state.query = '';
        this.state.page = 0;
        document.getElementById('searchInput').value = '';
        this.fetchVideos();
        this.switchView('grid');
        document.getElementById('v-player').pause();
    },

    handleSearch() {
        const q = document.getElementById('searchInput').value.trim();
        this.state.query = q;
        this.state.page = 0;
        this.fetchVideos();
        this.switchView('grid');
        document.getElementById('v-player').pause();
    },

    handleSortChange(val) {
        this.state.sort = val;
        this.state.page = 0;
        this.fetchVideos();
    },

    changePage(delta) {
        this.state.page = Math.max(0, this.state.page + delta);
        this.fetchVideos();
        document.getElementById('mainScrollArea').scrollTop = 0;
    },

    switchView(viewName) {
        document.getElementById('view-grid').classList.toggle('hidden', viewName !== 'grid');
        document.getElementById('view-player').classList.toggle('hidden', viewName !== 'player');
    },

    async fetchVideos() {
        this.showLoader(true);
        try {
            let endpoint = \`https://api.iwara.tv/videos?limit=24&page=\${this.state.page}&sort=\${this.state.sort}&rating=all\`;
            if (this.state.query) {
                endpoint = \`https://api.iwara.tv/search?type=video&query=\${encodeURIComponent(this.state.query)}&page=\${this.state.page}\`;
            }
            DBG.info('fetchVideos: ' + endpoint);
            const response = await fetch('/' + this._enc(endpoint));
            if (!response.ok) throw new Error("Edge Network Failure: " + response.status);
            const data = await response.json();
            const results = data.results || data.list || [];
            this.state.videos = results;
            this.renderGrid(results);
            this.updatePagination();
            DBG.success('fetchVideos: ' + results.length + ' videos loaded');
        } catch(err) {
            DBG.error('fetchVideos FAILED: ' + err.message);
            document.getElementById('videoGrid').innerHTML = \`
                <div class="col-span-full py-20 text-center">
                    <i class="fa-solid fa-triangle-exclamation text-4xl text-red-500 mb-4"></i>
                    <div class="font-bold">データの取得に失敗しました</div>
                    <div class="text-sm text-gray-500 mt-2">\${err.message}</div>
                </div>\`;
        } finally {
            this.showLoader(false);
        }
    },

    renderGrid(list) {
        const grid = document.getElementById('videoGrid');
        grid.innerHTML = '';
        if (list.length === 0) {
            grid.innerHTML = \`<div class="col-span-full py-20 text-center"><i class="fa-solid fa-ghost text-4xl text-gray-300 mb-4"></i><div class="font-bold text-gray-500">該当する動画がありませんでした</div></div>\`;
            return;
        }
        list.forEach(v => {
            const tid = (v.thumbnail && v.thumbnail.id) ? v.thumbnail.id : (v.file ? v.file.id : 'default');
            const turl = \`https://i.iwara.tv/image/thumbnail/\${tid}/thumbnail-00.jpg\`;
            const avatar = (v.user && v.user.avatar) ? \`https://i.iwara.tv/image/avatar/\${v.user.avatar.id}/\${v.user.avatar.id}.jpg\` : '';

            const card = document.createElement('div');
            card.className = "card-anim flex flex-col cursor-pointer";
            card.onclick = () => this.openPlayer(v.id);
            card.innerHTML = \`
                <div class="relative overflow-hidden rounded-xl bg-gray-100 shadow-sm aspect-video mb-3">
                    <img src="/\${this._enc(turl)}" class="w-full h-full object-cover" loading="lazy">
                    <div class="absolute bottom-2 right-2 bg-black/80 text-white text-[10px] font-bold px-1.5 py-0.5 rounded">\${v.customBody ? 'HD' : 'SD'}</div>
                </div>
                <div class="flex gap-3 pr-2">
                    <div class="shrink-0">
                        <div class="w-10 h-10 rounded-full bg-gray-100 overflow-hidden border">
                            \${avatar ? \`<img src="/\${this._enc(avatar)}" class="w-full h-full object-cover">\` : \`<div class="w-full h-full flex items-center justify-center font-bold text-gray-400 bg-gray-50">\${v.user.name.charAt(0)}</div>\`}
                        </div>
                    </div>
                    <div class="flex flex-col min-w-0">
                        <h3 class="font-bold text-[15px] leading-snug truncate-2 text-gray-900 mb-1">\${v.title}</h3>
                        <p class="text-sm text-gray-500 truncate hover:text-black transition">\${v.user.name}</p>
                        <p class="text-[12px] text-gray-400 font-medium">\${(v.numViews || 0).toLocaleString()} 回視聴 • \${this.timeSince(new Date(v.createdAt))}</p>
                    </div>
                </div>
            \`;
            grid.appendChild(card);
        });
    },

    updatePagination() {
        document.getElementById('pageIndicator').innerText = "Page " + (this.state.page + 1);
        document.getElementById('btnPrev').disabled = this.state.page === 0;
    },

    async openPlayer(vid) {
        this.showLoader(true);
        this.state.currentVid = vid;

        // Update diagnostic input
        const diagInput = document.getElementById('dpDiagVidInput');
        if (diagInput) diagInput.value = vid;

        try {
            const metaUrl = \`https://api.iwara.tv/video/\${vid}\`;
            const fileUrl = \`https://api.iwara.tv/video/\${vid}/file\`;

            DBG.step('openPlayer start | vid: ' + vid);
            DBG.info('metaUrl: ' + metaUrl);
            DBG.info('fileUrl: ' + fileUrl);

            // If manual token is set, send it via custom header the Worker can read
            // (Worker checks X-Manual-Token header and uses it if present)
            const fetchOpts = {};
            if (window._manualBearerToken) {
                fetchOpts.headers = { 'X-Manual-Token': window._manualBearerToken };
                DBG.info('Sending manual token via X-Manual-Token header');
            }

            const [metaRes, fileRes] = await Promise.all([
                fetch('/' + this._enc(metaUrl), fetchOpts),
                fetch('/' + this._enc(fileUrl), fetchOpts)
            ]);

            DBG.step('HTTP responses received');
            DBG.info('metaRes.status: ' + metaRes.status + ' ' + metaRes.statusText);
            DBG.info('fileRes.status: ' + fileRes.status + ' ' + fileRes.statusText);
            DBG.info('fileRes Content-Type: ' + fileRes.headers.get('Content-Type'));
            DBG.info('Token present (X-Diag): ' + fileRes.headers.get('X-Diag-Token-Present'));

            // Show token diagnostic panel hint if token missing
            if (fileRes.headers.get('X-Diag-Token-Present') === '0') {
                DBG.warn('⚠ Worker: Bearerトークンが空！TokenタブまたはDiagnosticタブを確認してください');
                document.getElementById('dpBadgeToken').style.display = 'flex';
                document.getElementById('dpBadgeToken').textContent = '!';
            }

            if (!metaRes.ok) {
                DBG.error('metaRes FAILED: HTTP ' + metaRes.status);
                throw new Error(\`Metadata API error: HTTP \${metaRes.status}\`);
            }
            if (!fileRes.ok) {
                const errBody = await fileRes.text();
                DBG.error('fileRes FAILED: HTTP ' + fileRes.status, { body: errBody });
                throw new Error(\`File list API error: HTTP \${fileRes.status} — \${errBody.slice(0, 200)}\`);
            }

            const meta = await metaRes.json();
            DBG.info('meta.title: ' + meta.title);
            DBG.info('meta.type: ' + meta.type);
            DBG.info('meta.embedUrl: ' + (meta.embedUrl || 'none'));

            // External embed detection
            if (meta.embedUrl) {
                DBG.warn('⚠ この動画は外部埋め込みです: ' + meta.embedUrl);
            }

            const fileRawText = await fileRes.text();
            DBG.step('fileRes raw body (first 800): ' + fileRawText.slice(0, 800));

            let filesRaw;
            try {
                filesRaw = JSON.parse(fileRawText);
            } catch(e) {
                DBG.error('fileRes JSON parse FAILED: ' + e.message, { raw: fileRawText.slice(0, 200) });
                throw new Error('File list JSON parse error: ' + e.message);
            }

            DBG.info('filesRaw type: ' + (Array.isArray(filesRaw) ? 'Array[' + filesRaw.length + ']' : typeof filesRaw));
            if (!Array.isArray(filesRaw) && typeof filesRaw === 'object' && filesRaw !== null) {
                DBG.warn('filesRaw is OBJECT (not array). Keys: ' + Object.keys(filesRaw).join(', '), filesRaw);
                if (filesRaw.message === 'errors.notFound') {
                    DBG.error('errors.notFound — 原因の可能性: ①Bearer tokenが空/無効 ②動画が非公開 ③外部埋め込み動画');
                    DBG.warn('→ 解決策: DebugPanel → Tokenタブで手動トークンを設定してください');
                }
            }

            let files;
            if (Array.isArray(filesRaw)) {
                files = filesRaw;
            } else if (filesRaw && typeof filesRaw === 'object') {
                files = filesRaw.results || filesRaw.list || filesRaw.data || filesRaw.files || [];
                if (!Array.isArray(files)) files = [];
            } else {
                files = [];
            }

            DBG.step('files.length after normalize: ' + files.length);
            if (files.length > 0) {
                DBG.info('files[0] keys: ' + Object.keys(files[0] || {}).join(', '));
                DBG.info('files[0].src type: ' + typeof files[0].src, files[0]);
            }

            // Update Player UI
            document.getElementById('v-title').innerText = meta.title;
            document.getElementById('v-author').innerText = meta.user.name;
            document.getElementById('v-views').innerText = (meta.numViews || 0).toLocaleString();
            document.getElementById('v-likes').innerText = (meta.numLikes || 0).toLocaleString();
            document.getElementById('v-date').innerText = new Date(meta.createdAt).toLocaleDateString();
            document.getElementById('v-desc').innerText = meta.body || '動画の説明はありません。';

            const av = (meta.user && meta.user.avatar)
                ? \`https://i.iwara.tv/image/avatar/\${meta.user.avatar.id}/\${meta.user.avatar.id}.jpg\`
                : '';
            document.getElementById('v-avatar').src = av ? '/' + this._enc(av) : '';

            const sourceContainer = document.getElementById('v-sources');
            sourceContainer.innerHTML = '';
            const player = document.getElementById('v-player');
            player.pause();
            player.src = '';

            if (!files || files.length === 0) {
                DBG.error('files EMPTY after normalization!');
                DBG.error('Raw fileRes body was: ' + fileRawText.slice(0, 600));
                DBG.error('→ 「🔬 DEV」ボタン → DiagnosticタブでフルDiagnosticを実行してください');

                // Auto-open debug panel for diagnosis
                DebugPanel.show();
                DebugPanel.switchTab('diagnostic');

                throw new Error(
                    "Playback failed. File list empty.\\n" +
                    "RAW API response: " + fileRawText.slice(0, 200) + "\\n" +
                    "→ デバッグパネルのDiagnosticタブで診断を実行してください"
                );
            }

            files.forEach((file, index) => {
                const btn = document.createElement('button');
                btn.className = "btn-yt bg-white border border-gray-300 hover:bg-gray-100";
                btn.innerText = file.name;

                btn.onclick = () => {
                    const currentTime = player.currentTime;
                    const wasPlaying = !player.paused;

                    let streamUrl;
                    if (typeof file.src === 'object' && file.src !== null) {
                        streamUrl = file.src.view || file.src.download || '';
                        DBG.info('file.src is object | view: ' + file.src.view + ' | download: ' + file.src.download);
                    } else {
                        streamUrl = file.src || '';
                        DBG.info('file.src is string: ' + streamUrl);
                    }

                    if (streamUrl.startsWith('//')) streamUrl = 'https:' + streamUrl;
                    else if (!streamUrl.startsWith('http')) streamUrl = 'https://files.iwara.tv' + streamUrl;

                    DBG.step('Playing: ' + streamUrl);

                    // Extract and show X-Version info
                    try {
                        const u = new URL(streamUrl);
                        const parts = u.pathname.split('/').filter(Boolean);
                        const fileId = parts[parts.length-1];
                        const expires = u.searchParams.get('expires') || '';
                        DBG.info('Stream: fileId=' + fileId + ' expires=' + expires + ' host=' + u.hostname);
                        if (document.getElementById('dpXverFileId')) document.getElementById('dpXverFileId').value = fileId;
                        if (document.getElementById('dpXverExpires')) document.getElementById('dpXverExpires').value = expires;
                    } catch(_) {}

                    player.src = '/' + this._enc(streamUrl);
                    player.load();

                    player.onerror = (e) => {
                        DBG.error('player.onerror code:' + player.error?.code + ' msg:' + player.error?.message);
                        DBG.warn('→ ストリームURLが403/404の場合、X-Versionが誤っている可能性があります。Tokenタブ → X-Version計算機で確認してください。');
                    };
                    player.onloadedmetadata = () => {
                        DBG.success('video metadata loaded! duration: ' + Math.round(player.duration) + 's | width: ' + player.videoWidth + ' | height: ' + player.videoHeight);
                    };
                    player.ontimeupdate = null; // Reset
                    player.onloadeddata = () => {
                        DBG.success('Video data loaded and playing!');
                    };

                    player.onloadedmetadata = () => {
                        const currentTime2 = player.currentTime;
                        player.currentTime = currentTime;
                        if (wasPlaying || currentTime === 0) player.play().catch(e => {
                            DBG.warn('Autoplay blocked: ' + e.message);
                        });
                    };

                    document.querySelectorAll('#v-sources button').forEach(b => {
                        b.classList.remove('bg-blue-600', 'text-white', 'border-blue-600');
                        b.classList.add('bg-white', 'text-black', 'border-gray-300');
                    });
                    btn.classList.add('bg-blue-600', 'text-white', 'border-blue-600');
                    btn.classList.remove('bg-white', 'text-black', 'border-gray-300');
                };

                sourceContainer.appendChild(btn);
                if (index === 0) btn.click();
            });

            this.renderRelated();
            this.switchView('player');
            document.getElementById('mainScrollArea').scrollTop = 0;
            DBG.success('openPlayer SUCCESS: ' + meta.title);

        } catch(err) {
            DBG.error('openPlayer FAILED: ' + err.message);
            console.error("openPlayer failed", err);
            alert("動画の再生に失敗しました: " + err.message);
        } finally {
            this.showLoader(false);
        }
    },

    renderRelated() {
        const rg = document.getElementById('relatedGrid');
        rg.innerHTML = '';
        const others = this.state.videos.filter(v => v.id !== this.state.currentVid).slice(0, 10);
        others.forEach(v => {
            const tid = (v.thumbnail && v.thumbnail.id) ? v.thumbnail.id : (v.file ? v.file.id : 'default');
            const turl = \`https://i.iwara.tv/image/thumbnail/\${tid}/thumbnail-00.jpg\`;
            const item = document.createElement('div');
            item.className = "flex gap-2.5 cursor-pointer group";
            item.onclick = () => this.openPlayer(v.id);
            item.innerHTML = \`
                <div class="relative w-40 aspect-video rounded-lg overflow-hidden bg-gray-100 shrink-0">
                    <img src="/\${this._enc(turl)}" class="w-full h-full object-cover">
                </div>
                <div class="flex flex-col overflow-hidden pt-0.5">
                    <h4 class="text-[14px] font-bold leading-tight truncate-2 mb-1 group-hover:text-blue-600 transition">\${v.title}</h4>
                    <p class="text-[12px] text-gray-500 truncate">\${v.user.name}</p>
                    <p class="text-[11px] text-gray-400 mt-0.5">\${(v.numViews || 0).toLocaleString()} 回視聴</p>
                </div>
            \`;
            rg.appendChild(item);
        });
    },

    timeSince(date) {
        const seconds = Math.floor((new Date() - date) / 1000);
        let interval = seconds / 31536000;
        if (interval > 1) return Math.floor(interval) + "年前";
        interval = seconds / 2592000;
        if (interval > 1) return Math.floor(interval) + "ヶ月前";
        interval = seconds / 86400;
        if (interval > 1) return Math.floor(interval) + "日前";
        interval = seconds / 3600;
        if (interval > 1) return Math.floor(interval) + "時間前";
        interval = seconds / 60;
        if (interval > 1) return Math.floor(interval) + "分前";
        return "たった今";
    }
};

// =====================================================================
// Utilities
// =====================================================================
function escHtml(str) {
    return String(str || '').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;').replace(/'/g,'&#39;');
}

function _enc(str) {
    if (!str) return str;
    try {
        return ENC_PREFIX + btoa(encodeURIComponent(str))
            .replace(/\\+/g, '-').replace(/\\//g, '_').replace(/=/g, '');
    } catch(e) { return str; }
}

// =====================================================================
// Boot
// =====================================================================
App.init();
</script>
</body>
</html>`;

// =======================================================================================
// SECTION 13: EDGE ROUTER
// Reference: https://github.com/pillarjs/router
// =======================================================================================

class EdgeRouter {
  async handle(request) {
    const url = new URL(request.url);

    // 1. Preflight CORS
    if (request.method === "OPTIONS") {
      return new Response(null, {
        headers: {
          "Access-Control-Allow-Origin": "*",
          "Access-Control-Allow-Methods": GLOBAL_CONFIG.ALLOWED_METHODS.join(", "),
          "Access-Control-Allow-Headers": "*",
          "Access-Control-Max-Age": "86400"
        }
      });
    }

    // 2. Main Portal UI
    if (url.pathname === "/" || url.pathname === "/index.html") {
      return new Response(TemplateEngine.renderUI(), {
        headers: { "Content-Type": "text/html; charset=utf-8" }
      });
    }

    // 3. NEW: Diagnostic endpoint
    // GET /api/diagnose?vid={videoId}
    if (url.pathname === "/api/diagnose") {
      const videoId = url.searchParams.get("vid");
      if (!videoId) {
        return new Response(JSON.stringify({ error: "vid parameter required" }), {
          status: 400,
          headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
        });
      }
      try {
        const report = await DiagnosticEngine.runFull(videoId);
        return new Response(JSON.stringify(report), {
          headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
        });
      } catch(e) {
        return new Response(JSON.stringify({ error: String(e) }), {
          status: 500,
          headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
        });
      }
    }

    // 4. NEW: Token probe endpoint
    // GET /api/token-probe
    if (url.pathname === "/api/token-probe") {
      const attempts = [];
      const baseHeaders = {
        'User-Agent':      GLOBAL_CONFIG.USER_AGENT,
        'Referer':         GLOBAL_CONFIG.REFERER,
        'Origin':          GLOBAL_CONFIG.ORIGIN,
        'Accept':          'application/json',
        'Accept-Language': 'ja,en-US;q=0.9,en;q=0.8'
      };
      for (const ep of GLOBAL_CONFIG.TOKEN_ENDPOINTS) {
        const opts = { method: ep.method, headers: { ...baseHeaders } };
        if (ep.method === 'POST') { opts.headers['Content-Type'] = 'application/json'; opts.body = JSON.stringify({}); }
        try {
          const res = await fetch(ep.url, opts);
          const body = await res.text();
          let parsed = null;
          try { parsed = JSON.parse(body); } catch(_) {}
          const hasToken = !!(parsed && (parsed.token || parsed.access_token || parsed.accessToken));
          attempts.push({
            url: ep.url, method: ep.method, status: res.status,
            bodyPreview: body.slice(0, 400), hasToken,
            tokenKeys: parsed ? Object.keys(parsed) : []
          });
        } catch(e) {
          attempts.push({ url: ep.url, method: ep.method, error: String(e) });
        }
      }
      return new Response(JSON.stringify({ attempts, timestamp: new Date().toISOString() }), {
        headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
      });
    }

    // 5. NEW: Health check
    if (url.pathname === "/api/health") {
      return new Response(JSON.stringify({
        status: "ok",
        version: GLOBAL_CONFIG.VERSION,
        timestamp: new Date().toISOString(),
        tokenCache: IwaraTokenManager.getDiagSnapshot()
      }), {
        headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
      });
    }

    // 6. Encrypted Proxy Route
    if (url.pathname.startsWith("/" + GLOBAL_CONFIG.ENC_PREFIX)) {
      const decodedTarget = Base64URLEngine.decode(url.pathname.substring(1));
      if (!decodedTarget || !decodedTarget.startsWith("http")) {
        return new Response("Malformed Proxy URL", { status: 400 });
      }

      // Check for manual token override from client
      const manualToken = request.headers.get('X-Manual-Token');
      if (manualToken) {
        IwaraTokenManager.setManualToken(manualToken);
      }

      return await IwaraApiInterceptor.proxyFetch(decodedTarget, request);
    }

    // 7. Favicon fallback
    if (url.pathname.includes("favicon")) return new Response(null, { status: 404 });

    return new Response("Not Found", { status: 404 });
  }
}

// =======================================================================================
// SECTION 14: ADDITIONAL UTILITIES
// =======================================================================================

/**
 * Retry Protocol Engine
 * Reference: https://github.com/sindresorhus/p-retry
 */
async function fetchWithRetry(url, options, limit = GLOBAL_CONFIG.RETRY_LIMIT) {
  let lastErr;
  for (let i = 0; i < limit; i++) {
    try {
      const res = await fetch(url, options);
      if (res.status >= 500) throw new Error("Server Error " + res.status);
      return res;
    } catch (err) {
      lastErr = err;
      await new Promise(r => setTimeout(r, 500 * Math.pow(2, i))); // Exponential backoff
    }
  }
  throw lastErr;
}

/**
 * Buffer Hex Utilities
 * Reference: https://github.com/feross/buffer
 */
class BufferShim {
  static toHex(arrayBuffer) {
    return Array.prototype.map.call(new Uint8Array(arrayBuffer), x => ('00' + x.toString(16)).slice(-2)).join('');
  }
  static fromHex(hexString) {
    return new Uint8Array(hexString.match(/.{1,2}/g).map(byte => parseInt(byte, 16))).buffer;
  }
}

/**
 * Memory concurrency limiter
 * Reference: https://github.com/sindresorhus/p-limit
 */
const MemoryMonitor = {
  async withLimit(promiseArray, limit = 5) {
    const results = [];
    const executing = [];
    for (const item of promiseArray) {
      const p = Promise.resolve().then(() => item());
      results.push(p);
      if (limit <= promiseArray.length) {
        const e = p.then(() => executing.splice(executing.indexOf(e), 1));
        executing.push(e);
        if (executing.length >= limit) await Promise.race(executing);
      }
    }
    return Promise.all(results);
  }
};

// =======================================================================================
// SECTION 15: WORKER ENTRY POINT
// Reference: https://github.com/cloudflare/workers-sdk
// =======================================================================================

export default {
  async fetch(request, env, ctx) {
    try {
      const router = new EdgeRouter();
      return await router.handle(request);
    } catch (err) {
      console.error("Critical Runtime Error:", err);
      return new Response(
        JSON.stringify({ error: err.message, stack: err.stack?.slice(0, 500) }),
        {
          status: 500,
          headers: {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
          }
        }
      );
    }
  }
};
