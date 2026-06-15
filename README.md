# GitHub-API-explorer-fetch-wrapper-bilan-
GitHub API bilan ishlash, so'rovlarni boshqarish, asinxron generator yordamida sahifalash (pagination) va rate-limit tahlilini amalga oshiruvchi universal va xavfsiz GitHub Dashboard Engine moduli.

🛠️ 1. Universal API Wrapper va GitHub Engine (githubEngine.js)
JavaScript
// Konstantalar (Xavfsizlik uchun token va sozlamalar)
const GITHUB_TOKEN = ""; // Private repolar uchun "Bearer <token>" qo'yish mumkin
const BASE_URL = "https://api.github.com";

const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));

// Universal API Wrapper
export const api = async (url, { options = {}, timeout = 5000, retries = 3 } = {}) => {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeout);

  const defaultHeaders = {
    "Accept": "application/vnd.github+json",
    "X-GitHub-Api-Version": "2022-11-28",
    ...(GITHUB_TOKEN ? { "Authorization": GITHUB_TOKEN } : {})
  };

  const config = {
    ...options,
    headers: { ...defaultHeaders, ...options.headers },
    signal: controller.signal
  };

  let attempt = 0;
  while (attempt < retries) {
    try {
      attempt++;
      const response = await fetch(url, config);
      clearTimeout(id);

      // Rate Limit tahlili
      const rateLimit = {
        limit: response.headers.get("x-ratelimit-limit"),
        remaining: response.headers.get("x-ratelimit-remaining"),
        reset: new Date(response.headers.get("x-ratelimit-reset") * 1000).toLocaleTimeString()
      };

      if (!response.ok) {
        if (response.status === 403 && rateLimit.remaining === "0") {
          throw new Error(`Rate Limit tugadi. Blokdan chiqish vaqti: ${rateLimit.reset}`);
        }
        throw new Error(`Xatolik: ${response.status} - ${response.statusText}`);
      }

      const data = await response.json();
      return { data, rateLimit };

    } catch (error) {
      if (error.name === "AbortError") throw new Error(`So'rov vaqti tugadi (Timeout: ${timeout}ms)`);
      if (attempt >= retries) throw error;
      
      console.warn(`⚠️ Ulanish muvaffaqiyatsiz. Qayta urinish #${attempt}...`);
      await delay(500);
    }
  }
};

// Asinxron Generator: Pagination bilan barcha repolarni olish
export async function* fetchAllRepos(username) {
  let page = 1;
  const perPage = 30; // Har bir sahifadagi elementlar soni

  while (true) {
    const url = `${BASE_URL}/users/${username}/repos?page=${page}&per_page=${perPage}&sort=updated`;
    const { data } = await api(url);

    if (data.length === 0) break; // Boshqa repo qolmagan bo'lsa to'xtaydi

    for (const repo of data) {
      yield {
        name: repo.name,
        stars: repo.stargazers_count,
        forks: repo.forks_count,
        language: repo.language ?? "Noma'lum"
      };
    }
    page++;
  }
}
📊 2. Ma'lumotlarni yig'ish va Jadval ko'rinishida chiqarish (app.js)
JavaScript
import { api, fetchAllRepos } from './githubEngine.js';

const showDashboard = async (username) => {
  try {
    console.log(`🔍 ${username} profili yuklanmoqda...`);
    
    // 1. User Profile Ma'lumotlarini olish
    const profileUrl = `https://api.github.com/users/${username}`;
    const { data: profile, rateLimit } = await api(profileUrl);

    console.log(`\n================ PROFILE INFO ================`);
    console.log(`Mijoz: ${profile.name ?? profile.login}`);
    console.log(`Biografiya: ${profile.bio ?? "Mavjud emas"}`);
    console.log(`Public Repolar soni: ${profile.public_repos} ta`);
    console.log(`----------------------------------------------`);
    console.log(`[Rate Limit] Jami: ${rateLimit.limit} | Qoldi: ${rateLimit.remaining} | Tiklanish: ${rateLimit.reset}`);
    console.log(`==============================================\n`);

    // 2. Pagination bilan barcha repolarni Generator orqali yig'ish
    console.log("📦 Repozitoriyalar jadvali shakllantirilmoqda...\n");
    
    const headerName = "Repo Nomi".padEnd(25);
    const headerStars = "Stars".padStart(8);
    const headerForks = "Forks".padStart(8);
    const headerLang = "Til".padStart(15);
    const line = "-".repeat(60);

    console.log(line);
    console.log(`${headerName}${headerStars}${headerForks}${headerLang}`);
    console.log(line);

    let repoCount = 0;
    // Async generator ustida for-await-of sikli
    for await (const repo of fetchAllRepos(username)) {
      repoCount++;
      const name = repo.name.slice(0, 23).padEnd(25);
      const stars = String(repo.stars).padStart(8);
      const forks = String(repo.forks).padStart(8);
      const lang = repo.language.slice(0, 14).padStart(15);
      
      console.log(`${name}${stars}${forks}${lang}`);
      
      // Cheksiz davom etmasligi uchun (masalan, dastlabki 15 tasini chiqarish)
      if (repoCount >= 15) break; 
    }
    
    console.log(line);
    console.log(`Ko'rsatilgan repolar: ${repoCount} ta`);
    console.log(line);

  } catch (error) {
    console.error("❌ Dashboard xatoligi:", error.message);
  }
};

// Sinab ko'rish (Masalan, "gaearon" — Dan Abramov profili)
showDashboard("gaearon");
💎 Ushbu arxitekturaning afzalliklari:
Xotirani tejash (Async Generator): Yuzlab repolarni bir vaqtda ulkan bitta massivga yuklab xotirani to'ldirmaydi. yield yordamida sahifalarni bittalab yuklab, oqim (stream) ko'rinishida taqdim etadi.

Xavfsiz AbortController: Agar GitHub serverlari 5 soniya ichida (yoki belgilangan vaqtda) javob bermasa, so'rov avtomatik ravishda uziladi va tarmoq osilib qolmaydi.

Aqlli Rate-Limit Nazorati: Har bir so'rovda GitHub qaytargan limit headerlari tekshiriladi. Limit tugagan bo'lsa, kod xatolik berishdan avval aniq qachon blokdan chiqish vaqtini ko'rsatadi.
