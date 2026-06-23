// ==================================================
// 淘宝/天猫历史价格比价脚本 v1.2
// 适用工具: Quantumult X
// 功能: 拦截商品详情接口，查询历史最低价并发送通知
// 数据来源: 慢慢买 (manmanbuy.com)
// ==================================================

"use strict";

// ─── 工具函数 ────────────────────────────────────────

/**
 * 从 URL 中提取商品 ID
 */
function getItemIdFromUrl(url) {
  const patterns = [
    /[?&]id=(\d+)/,
    /[?&]itemId=(\d+)/,
    /[?&]item_id=(\d+)/,
    /\/item\/(\d+)\.html/,
    /\/(\d{10,})\?/,
  ];
  for (const pattern of patterns) {
    const match = url.match(pattern);
    if (match) return match[1];
  }
  return null;
}

/**
 * 格式化价格，保留两位小数
 */
function fmt(price) {
  return parseFloat(price).toFixed(2);
}

/**
 * 根据当前价与历史价给出购买建议
 */
function getAdvice(current, min, max) {
  const cur = parseFloat(current);
  const low = parseFloat(min);
  const high = parseFloat(max);
  const ratio = (cur - low) / (high - low || 1); // 在区间内的位置 0~1

  if (cur <= low * 1.02) return "🔥 近历史最低！强烈推荐入手";
  if (cur <= low * 1.10) return "✅ 价格较低，可以考虑入手";
  if (ratio < 0.4)       return "👍 价格偏低，时机不错";
  if (ratio < 0.6)       return "ℹ️ 价格处于正常区间";
  if (ratio < 0.8)       return "⚠️ 价格偏高，建议再等等";
  return                        "🚫 接近历史最高，不推荐现在购买";
}

// ─── 主逻辑 ──────────────────────────────────────────

const reqUrl  = $request.url;
const resBody = $response.body;

let itemId       = null;
let itemTitle    = "";
let currentPrice = "";

// 1. 解析响应 JSON，提取商品基本信息
try {
  const json = JSON.parse(resBody);
  const data = json.data || {};

  // 格式 A：mtop.taobao.detail.getdetail / getdetail2
  if (data.item && data.item.itemId) {
    itemId       = String(data.item.itemId);
    itemTitle    = data.item.title || "";
    currentPrice = (data.price && data.price.price) ? String(data.price.price) : "";
  }

  // 格式 B：apiStack 嵌套
  if (!itemId && Array.isArray(data.apiStack) && data.apiStack[0]) {
    try {
      const inner     = JSON.parse(data.apiStack[0].value || "{}");
      const innerData = inner.data || {};
      if (innerData.itemDO) {
        itemId    = String(innerData.itemDO.itemId || "");
        itemTitle = innerData.itemDO.title || "";
      }
      if (innerData.priceVO) {
        currentPrice = String(innerData.priceVO.price || "");
      }
    } catch (_) {}
  }

  // 格式 C：直接字段
  if (!itemId && data.itemId) {
    itemId    = String(data.itemId);
    itemTitle = data.title || "";
  }

} catch (_) {
  // 响应体不是 JSON，继续尝试从 URL 获取 ID
}

// 2. 兜底：从请求 URL 提取商品 ID
if (!itemId) {
  itemId = getItemIdFromUrl(reqUrl);
}

// 没有商品 ID 则直接放行
if (!itemId) {
  $done({ body: resBody });
  return; // eslint-disable-line no-unreachable
}

// 3. 请求慢慢买历史价格数据
const apiUrl = `https://www.manmanbuy.com/getcjson.aspx?action=getprice&itemid=${itemId}&p=tb`;

$task.fetch({
  url: apiUrl,
  method: "GET",
  headers: {
    "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148",
    "Referer":    "https://www.manmanbuy.com/",
    "Accept":     "application/json, text/javascript, */*; q=0.01"
  }
}).then(resp => {

  // ── 组装通知 ──
  const notifyTitle    = "🛒 淘宝比价助手";
  const displayTitle   = itemTitle.length > 22
    ? itemTitle.substring(0, 22) + "…"
    : (itemTitle || `商品 ID: ${itemId}`);
  let notifyBody = "";

  try {
    const pd = JSON.parse(resp.body);

    if (pd && Array.isArray(pd.pricelist) && pd.pricelist.length > 0) {
      // 提取并过滤有效价格
      const prices = pd.pricelist
        .map(p => parseFloat(p.price))
        .filter(p => !isNaN(p) && p > 0);

      if (prices.length > 0) {
        const minPrice  = Math.min(...prices);
        const maxPrice  = Math.max(...prices);
        const lastPrice = prices[prices.length - 1];
        const curStr    = currentPrice || fmt(lastPrice);

        // 近 30 天均价（取最近 30 个数据点）
        const recent30  = prices.slice(-30);
        const avg30     = recent30.reduce((s, v) => s + v, 0) / recent30.length;

        notifyBody = [
          `💰 当前价格：¥${curStr}`,
          `📉 历史最低：¥${fmt(minPrice)}`,
          `📈 历史最高：¥${fmt(maxPrice)}`,
          `📊 近30天均：¥${fmt(avg30)}`,
          `🔢 价格记录：${prices.length} 条`,
          ``,
          getAdvice(curStr, minPrice, maxPrice)
        ].join("\n");

      } else {
        notifyBody = "⚠️ 无法解析价格数据";
      }
    } else {
      notifyBody = "📭 暂无历史价格记录\n可能是新品或数据库未收录";
    }
  } catch (e) {
    notifyBody = `❌ 数据解析失败：${e.message}`;
  }

  $notify(notifyTitle, displayTitle, notifyBody);
  $done({ body: resBody });

}).catch(err => {
  $notify("淘宝比价助手", "⚠️ 网络请求失败", err.message || "请检查网络或稍后重试");
  $done({ body: resBody });
});
