# Spotify 每日推荐 & 私人雷达 接入指南

> 本文档基于 Spotify Web API 官方文档（https://developer.spotify.com/documentation/web-api）编写。

---

## 一、功能概述

当前项目已有网易云的"每日推荐"和"私人雷达"功能（通过 `/api/discover/home` 接口）。本指南描述如何为 Spotify 用户新增同等功能，使已登录 Spotify 的用户也能在首页看到个性化推荐。

### 现有逻辑（网易云）

- **每日推荐**：调用网易云 `recommend_songs` 接口，返回最多 12 首推荐歌曲，直接加入播放队列
- **私人雷达**：调用网易云 `recommend_resource` 接口，返回推荐歌单，加载整个歌单到播放队列
- 前端入口：`playHomeDaily()` 和 `playHomePrivateRadio()` 函数（`public/index.html`）

### 改造目标

```
点击"每日推荐" →
  ├─ 已登录网易云 → 调用 /api/discover/home（不变）
  └─ 已登录 Spotify → 调用 /api/spotify/discover/home（新增）

点击"私人雷达" →
  ├─ 已登录网易云 → 调用 /api/discover/home（不变）
  └─ 已登录 Spotify → 调用 /api/spotify/discover/home（新增）
```

---

## 二、Spotify API 接口说明

### 2.1 获取用户最常听的歌曲（核心接口）

```
GET https://api.spotify.com/v1/me/top/{type}
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| type | string | 是 | `"artists"` 或 `"tracks"` |
| time_range | string | 否 | `"short_term"`（约4周）、`"medium_term"`（约6个月，默认）、`"long_term"`（约1年） |
| limit | integer | 否 | 返回数量，默认 20，范围 1-50 |
| offset | integer | 否 | 分页偏移，默认 0 |

**所需 Scope**：`user-top-read`

**返回字段（type=tracks 时）**：每首歌包含 `id`、`name`、`duration_ms`、`explicit`、`artists[]`（含 `id`、`name`）、`album`（含 `name`、`images[]`）、`uri` 等。

**用途**：作为"私人雷达"的核心数据源——用户最常听的歌曲代表其口味偏好。

---

### 2.2 获取推荐歌曲

```
GET https://api.spotify.com/v1/recommendations
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| seed_artists | string | 否 | 逗号分隔的 Spotify 艺人 ID（最多5个种子，三种种子合计不超过5） |
| seed_tracks | string | 否 | 逗号分隔的 Spotify 歌曲 ID |
| seed_genres | string | 否 | 逗号分隔的风格标签 |
| limit | integer | 否 | 返回数量，默认 20，范围 1-100 |
| market | string | 否 | ISO 3166-1 alpha-2 国家代码 |

**可调参数**（每个支持 min_/max_/target_ 前缀）：

| 参数 | 范围 | 说明 |
|------|------|------|
| acousticness | 0-1 | 原声程度 |
| danceability | 0-1 | 可舞性 |
| energy | 0-1 | 能量感 |
| instrumentalness | 0-1 | 纯器乐程度 |
| popularity | 0-100 | 热门程度 |
| tempo | - | BPM |
| valence | 0-1 | 音乐积极性/愉悦度 |

**所需 Scope**：无额外 scope（使用已有的 access_token 即可）

**⚠️ 重要**：该接口已被标记为 **Deprecated**（2026年废弃），但截至目前仍可正常使用。后续 Spotify 可能推出替代方案。

**返回字段**：`tracks[]` 数组，每首歌的字段与搜索结果一致（含 `id`、`name`、`artists`、`album`、`duration_ms` 等）。

**用途**：作为"每日推荐"的核心数据源——根据用户口味种子生成个性化推荐列表。

---

### 2.3 获取最近播放记录

```
GET https://api.spotify.com/v1/me/player/recently-played
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| limit | integer | 否 | 默认 20，范围 1-50 |
| after | integer | 否 | Unix 毫秒时间戳，获取此时间之后的记录 |
| before | integer | 否 | Unix 毫秒时间戳，获取此时间之前的记录 |

**所需 Scope**：`user-read-recently-played`

**返回字段**：`items[]` 数组，每项包含 `track`（完整歌曲对象）、`played_at`（播放时间）、`context`（播放来源）。

**用途**：补充推荐种子——最近播放的歌曲可以作为 `/recommendations` 的 seed_tracks 输入。

---

### 2.4 获取艺人热门歌曲

```
GET https://api.spotify.com/v1/artists/{id}/top-tracks
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | 是 | Spotify 艺人 ID |
| market | string | 否 | ISO 3166-1 alpha-2 国家代码 |

**所需 Scope**：无额外 scope

**返回字段**：`tracks[]` 数组，包含该艺人最热门的完整歌曲对象。

---

### 2.5 获取相似艺人

```
GET https://api.spotify.com/v1/artists/{id}/related-artists
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | 是 | Spotify 艺人 ID |

**所需 Scope**：无额外 scope

**返回字段**：`artists[]` 数组，包含相似艺人的完整信息（含 `id`、`name`、`genres`、`images` 等）。

---

## 三、需要新增的 OAuth Scope

当前项目已有的 Spotify Scopes（`server.js` 第 217-229 行）：

```javascript
const SPOTIFY_SCOPES = [
  'streaming',
  'user-read-email',
  'user-read-private',
  'user-read-playback-state',
  'user-modify-playback-state',
  'playlist-read-private',
  'playlist-read-collaborative',
  'playlist-modify-public',
  'playlist-modify-private',
  'user-library-modify',
  'user-library-read',
].join(' ');
```

**需要新增**：

```javascript
'user-top-read',            // 获取用户最常听的歌曲/艺人
'user-read-recently-played', // 获取最近播放记录
```

> ⚠️ 新增 scope 后，已登录用户需要**重新授权**才能使用新功能。可在前端检测到 scope 不足时提示用户重新登录。

---

## 四、后端实现（server.js）

### 4.1 新增接口处理函数

参考现有的 `handleDiscoverHome()`（第 1999 行），新增 `handleSpotifyDiscoverHome()` 函数：

```javascript
async function handleSpotifyDiscoverHome() {
  const info = await getSpotifyLoginInfo();
  if (!info.loggedIn) {
    return {
      loggedIn: false,
      user: null,
      dailySongs: [],
      playlists: [],
      mode: 'starter',
      updatedAt: Date.now(),
    };
  }

  try {
    // 1. 获取用户最常听的歌曲（短期，用于生成推荐种子）
    const topTracksData = await spotifyApiGet('/me/top/tracks', {
      time_range: 'short_term',
      limit: '10',
    });
    const topTrackIds = (topTracksData.items || []).map(t => t.id).filter(Boolean);

    // 2. 获取用户最常听的艺人（用于生成推荐种子）
    const topArtistsData = await spotifyApiGet('/me/top/artists', {
      time_range: 'medium_term',
      limit: '5',
    });
    const topArtistIds = (topArtistsData.items || []).map(a => a.id).filter(Boolean);

    // 3. 获取最近播放记录（补充种子）
    const recentData = await spotifyApiGet('/me/player/recently-played', {
      limit: '10',
    });
    const recentTrackIds = (recentData.items || []).map(it => it.track && it.track.id).filter(Boolean);

    // 4. 构建推荐种子（合计不超过5个）
    // 策略：取 2 个 top tracks + 2 个 top artists + 1 个最近播放
    const seedTracks = topTrackIds.slice(0, 2).concat(recentTrackIds.slice(0, 1)).join(',');
    const seedArtists = topArtistIds.slice(0, 2).join(',');

    // 5. 获取推荐歌曲
    const recParams = { limit: '20', market: 'from_token' };
    if (seedTracks) recParams.seed_tracks = seedTracks;
    if (seedArtists) recParams.seed_artists = seedArtists;
    // 如果种子不足，补充风格种子
    if (!seedTracks && !seedArtists) {
      recParams.seed_genres = 'pop,rock,hip-hop';
    }

    const recData = await spotifyApiGet('/recommendations', recParams);
    const dailySongs = (recData.tracks || [])
      .map(spotifyMapTrack)
      .filter(t => t.id);

    // 6. 构建"私人歌单"——将 top tracks 包装为一个虚拟歌单
    const topTracks = (topTracksData.items || [])
      .map(spotifyMapTrack)
      .filter(t => t.id);

    const playlists = [];
    if (topTracks.length) {
      playlists.push({
        provider: 'spotify',
        source: 'spotify',
        id: '__spotify_private_radio__',  // 特殊 ID，前端加载时识别
        name: '私人雷达',
        cover: topTracks[0].cover || '',
        trackCount: topTracks.length,
        creator: info.nickname || 'Spotify',
        _tracks: topTracks,  // 附带歌曲数据，避免二次请求
      });
    }

    return {
      loggedIn: true,
      user: { nickname: info.nickname || 'Spotify', avatar: info.avatar || '' },
      dailySongs,
      playlists,
      mode: 'member',
      updatedAt: Date.now(),
    };
  } catch (err) {
    console.error('[SpotifyDiscoverHome]', err);
    return {
      loggedIn: true,
      user: { nickname: info.nickname || 'Spotify', avatar: info.avatar || '' },
      dailySongs: [],
      playlists: [],
      mode: 'member',
      error: err.message,
      updatedAt: Date.now(),
    };
  }
}
```

### 4.2 新增 HTTP 路由

在 `server.js` 的路由处理区域（参考第 3986 行附近的 Spotify 路由），新增：

```javascript
if (pn === '/api/spotify/discover/home') {
  try { sendJSON(res, await handleSpotifyDiscoverHome()); }
  catch (err) { console.error('[SpotifyDiscoverHome]', err); sendJSON(res, { loggedIn: false, dailySongs: [], playlists: [], error: err.message }, 500); }
  return;
}
```

---

## 五、前端实现（public/index.html）

### 5.1 修改 playHomeDaily() 函数

当前逻辑（约第 16057 行）只调用网易云接口。需要改为按平台分流：

```javascript
async function playHomeDaily() {
  homeForcedOpen = false;
  homeSuppressed = false;
  setHomeControlsLocked(false);

  // 判断登录状态
  var hasNetease = typeof loginStatus !== 'undefined' && loginStatus.loggedIn;
  var hasSpotify = typeof spotifyLoginStatus !== 'undefined' && spotifyLoginStatus.loggedIn;

  if (!hasNetease && !hasSpotify) {
    showLoginModal({ source: 'home-daily' });
    return;
  }

  // Spotify 用户走 Spotify 接口
  if (hasSpotify && !hasNetease) {
    try {
      var r = await apiJson('/api/spotify/discover/home');
      if (r && r.dailySongs && r.dailySongs.length) {
        playQueue = r.dailySongs.map(cloneSong);
        currentIdx = 0;
        safeRenderQueuePanel('home-daily');
        safeShelfRebuild('home-daily', true);
        forcePlaybackControlsInteractive();
        playQueueAt(0).catch(function(e){ console.warn('[HomeDailySpotify]', e); });
        return;
      }
    } catch (e) {
      console.warn('[HomeDailySpotify]', e);
    }
    showToast('暂无推荐，请多听一些歌曲后再试');
    return;
  }

  // 网易云用户走原有逻辑（不变）
  await waitForHomeDiscoverIdle();
  if (!homeDiscoverState.loaded || (!homeDiscoverState.songs.length && !homeDiscoverState.loading)) {
    await loadHomeDiscover(true);
  }
  if (!homeDiscoverState.songs.length) {
    runHomeSearch('每日推荐');
    return;
  }
  playQueue = homeDiscoverState.songs.map(cloneSong);
  currentIdx = 0;
  safeRenderQueuePanel('home-daily');
  safeShelfRebuild('home-daily', true);
  forcePlaybackControlsInteractive();
  playQueueAt(0).catch(function(e){ console.warn('[HomeDailyPlay]', e); });
}
```

### 5.2 修改 playHomePrivateRadio() 函数

当前逻辑（约第 16080 行）同样需要按平台分流：

```javascript
async function playHomePrivateRadio() {
  homeForcedOpen = false;
  homeSuppressed = false;
  setHomeControlsLocked(false);

  var hasNetease = typeof loginStatus !== 'undefined' && loginStatus.loggedIn;
  var hasSpotify = typeof spotifyLoginStatus !== 'undefined' && spotifyLoginStatus.loggedIn;

  if (!hasNetease && !hasSpotify) {
    showLoginModal({ source: 'home-private' });
    return;
  }

  // Spotify 用户走 Spotify 接口
  if (hasSpotify && !hasNetease) {
    try {
      var r = await apiJson('/api/spotify/discover/home');
      if (r && r.playlists && r.playlists.length) {
        var pl = r.playlists[0];
        // 如果是特殊 ID 且自带歌曲数据，直接使用
        if (pl.id === '__spotify_private_radio__' && pl._tracks && pl._tracks.length) {
          playQueue = pl._tracks.map(cloneSong);
          currentIdx = 0;
          safeRenderQueuePanel('home-private-radio');
          safeShelfRebuild('home-private-radio', true);
          forcePlaybackControlsInteractive();
          playQueueAt(0).catch(function(e){ console.warn('[HomePrivateSpotify]', e); });
          return;
        }
        // 否则加载歌单
        await loadPlaylistIntoQueueById(pl.id, true, pl.name || '私人雷达');
        return;
      }
      // 没有推荐歌单，退回用推荐歌曲
      if (r && r.dailySongs && r.dailySongs.length) {
        playQueue = r.dailySongs.map(cloneSong);
        currentIdx = 0;
        safeRenderQueuePanel('home-private-radio');
        safeShelfRebuild('home-private-radio', true);
        forcePlaybackControlsInteractive();
        playQueueAt(0).catch(function(e){ console.warn('[HomePrivateSpotify]', e); });
        return;
      }
    } catch (e) {
      console.warn('[HomePrivateSpotify]', e);
    }
    showToast('暂无推荐，请多听一些歌曲后再试');
    return;
  }

  // 网易云用户走原有逻辑（不变）
  await waitForHomeDiscoverIdle();
  if (!homeDiscoverState.loaded || ((!homeDiscoverState.playlists.length && !homeDiscoverState.songs.length) && !homeDiscoverState.loading)) {
    await loadHomeDiscover(true);
  }
  if (homeDiscoverState.songs.length) {
    playQueue = homeDiscoverState.songs.map(cloneSong);
    currentIdx = 0;
    safeRenderQueuePanel('home-private-radio');
    safeShelfRebuild('home-private-radio', true);
    forcePlaybackControlsInteractive();
    playQueueAt(0).catch(function(e){ console.warn('[HomePrivatePlay]', e); });
    return;
  }
  var item = homeDiscoverState.playlists[0];
  if (item && item.id) {
    await loadPlaylistIntoQueueById(item.id, true, item.name || '私人雷达');
    return;
  }
  openHomeLibrary();
}
```

---

## 六、注意事项

### 6.1 2026 API 变更

- `GET /recommendations` 已标记为 **Deprecated**，目前仍可用，但需关注 Spotify 后续公告
- 部分 Playlist 接口已废弃旧版本，使用新版（参见项目中已有的 `/playlists/{id}/items` 调用）
- 存在 **2026 年 2 月 Dev Mode 迁移指南**，如遇权限问题需查阅

### 6.2 歌曲格式兼容

Spotify 歌曲需通过已有的 `spotifyMapTrack()` 函数（`server.js` 第 415 行）转换为统一格式：

```javascript
{
  provider: 'spotify',
  source: 'spotify',
  type: 'spotify',
  id: '歌曲ID',
  uri: 'spotify:track:歌曲ID',
  name: '歌曲名',
  artist: '艺人1 / 艺人2',
  artists: [{ id: 'xxx', name: 'xxx' }],
  artistId: '首个艺人ID',
  album: '专辑名',
  cover: '封面URL',
  duration: 240000,  // 毫秒
  explicit: false,
  previewUrl: '',
}
```

### 6.3 Scope 升级兼容

新增 scope 后，已有用户的 token 可能权限不足。建议：
- 在 `getSpotifyLoginInfo()` 中检测 token scope 是否包含 `user-top-read`
- 如不足，返回 `{ needReauth: true }`，前端提示用户重新登录
- 或者：推荐功能降级处理——scope 不足时跳过 Spotify 推荐，仅展示网易云推荐

### 6.4 推荐质量优化

推荐质量取决于种子选择。建议的策略：
- **每日推荐**：用 2 个短期 top tracks + 2 个中期 top artists + 1 个最近播放作为种子
- **私人雷达**：直接使用 top tracks 作为播放列表（无需调用 /recommendations）
- 可通过 `target_energy`、`target_valence` 等参数微调推荐风格
- 如果用户听歌记录太少（种子不足），使用 `seed_genres: 'pop'` 兜底

### 6.5 虚拟歌单 ID

"私人雷达"不是一个真实的 Spotify 歌单，而是由 top tracks 动态生成的。使用特殊 ID `__spotify_private_radio__` 标识，前端加载时需识别此 ID 并直接使用 `_tracks` 数据，而非调用歌单详情接口。

---

## 七、测试要点

1. **仅登录网易云**：每日推荐/私人雷达行为应与改造前完全一致
2. **仅登录 Spotify**：应调用新接口，返回推荐歌曲并可播放
3. **同时登录两个平台**：可选择展示哪个平台的推荐（建议优先网易云，因其推荐更成熟）
4. **未登录**：弹出登录框（不变）
5. **Spotify 无听歌记录**：应有兜底策略（genre seeds），不应报错
6. **Scope 不足**：应有优雅降级或重新授权提示
7. **推荐歌曲可正常播放**：确保通过 `spotifyMapTrack` 转换的歌曲能正常加入播放队列和播放
