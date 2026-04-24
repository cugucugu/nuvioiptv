const express = require("express");
const axios = require("axios");
const { addonBuilder, serveHTTP } = require("stremio-addon-sdk");

const PORT = process.env.PORT || 7000;

// 🔗 SENİN KAYNAKLAR
const SOURCES = [
  "http://hayattv.pepbox.xyz:8080/get.php?username=cml7015&password=WzzLDqprxqeq&type=m3u_plus&output=mpegts",
  "https://raw.githubusercontent.com/cugucugu/nuvioiptv/refs/heads/main/haber.m3u"
];

let cache = [];
let grouped = {};
let lastUpdate = 0;

// 🧠 PARSER
function parseM3U(data) {
  const lines = data.split("\n");
  const result = [];

  for (let i = 0; i < lines.length; i++) {
    if (lines[i].startsWith("#EXTINF")) {
      const info = lines[i];

      const name = info.split(",")[1] || "Unknown";
      const url = lines[i + 1];

      const logo = info.match(/tvg-logo="(.*?)"/);
      const group = info.match(/group-title="(.*?)"/);

      result.push({
        id: name.toLowerCase().replace(/\s/g, ""),
        name,
        url,
        poster: logo ? logo[1] : "",
        group: group ? group[1].toLowerCase() : "other"
      });
    }
  }

  return result;
}

// 🧹 TEMİZLE
function clean(list) {
  const seen = new Set();

  return list.filter(item => {
    const key = item.name.toLowerCase();
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });
}

// 📂 GRUPLA
function groupByCategory(streams) {
  const groups = {};

  streams.forEach(s => {
    const g = s.group || "other";
    if (!groups[g]) groups[g] = [];
    groups[g].push(s);
  });

  return groups;
}

// 🔄 VERİ YÜKLE
async function loadData() {
  let all = [];

  for (const src of SOURCES) {
    try {
      const res = await axios.get(src, { timeout: 8000 });
      all = all.concat(parseM3U(res.data));
    } catch (e) {
      console.log("Hata:", src);
    }
  }

  all = clean(all);
  cache = all;
  grouped = groupByCategory(all);
  lastUpdate = Date.now();
}

// ⚡ CACHE
async function getData() {
  if (Date.now() - lastUpdate > 15 * 60 * 1000) {
    await loadData();
  }
  return { cache, grouped };
}

// INIT
loadData();

// 🚀 ADDON
const builder = new addonBuilder({
  id: "com.kulkul.iptv",
  version: "1.0.0",
  name: "Kulkul IPTV"
});

// 📺 CATALOG
builder.defineCatalogHandler(async ({ id }) => {
  const { grouped } = await getData();

  if (!grouped[id]) return { metas: [] };

  return {
    metas: grouped[id].map(ch => ({
      id: ch.id,
      type: "tv",
      name: ch.name,
      poster: ch.poster
    }))
  };
});

// ▶️ STREAM
builder.defineStreamHandler(async ({ id }) => {
  const { cache } = await getData();
  const ch = cache.find(c => c.id === id);

  if (!ch) return { streams: [] };

  return {
    streams: [
      {
        title: ch.name,
        url: ch.url
      }
    ]
  };
});

// 🌐 MANIFEST (dinamik kategori)
builder.defineManifest(() => {
  const catalogs = Object.keys(grouped).map(g => ({
    type: "tv",
    id: g,
    name: g.toUpperCase()
  }));

  return {
    id: "com.cugucugu.iptv",
    version: "1.0.0",
    name: "CUGUCUGU IPTV",
    resources: ["catalog", "stream"],
    types: ["tv"],
    catalogs
  };
});

// 🌍 SERVER
const app = express();
serveHTTP(builder.getInterface(), { app });

app.listen(PORT, () => {
  console.log("Addon running on port", PORT);
});
