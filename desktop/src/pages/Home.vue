<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from "vue";
import { useRouter } from "vue-router";
import { useI18n } from "vue-i18n";
import { listen, type UnlistenFn } from "@tauri-apps/api/event";
import {
  NAlert,
  NLayout,
  NLayoutHeader,
  NLayoutContent,
  NSpace,
  NButton,
  NDropdown,
  NModal,
  NTag,
  NText,
  NEmpty,
  NScrollbar,
  NSpin,
  useDialog,
  useMessage,
} from "naive-ui";
import { check } from "@tauri-apps/plugin-updater";
import { relaunch } from "@tauri-apps/plugin-process";
import { platform } from "@tauri-apps/plugin-os";
import type { DropdownOption } from "naive-ui";
import { useAuthStore } from "@/stores/auth";
import { useThemeStore } from "@/stores/theme";
import { useConnectionStore } from "@/stores/connection";
import { usePlanStore } from "@/stores/plan";
import { formatError } from "@/utils/error";
import { api } from "@/api";
import type {
  ConnectStage,
  HelperStatus,
  KernelHealth,
  KernelVersion,
  NodeGeo,
  ProxyGroup,
  TunnelMode,
} from "@/types";
import WorldMap, { type NodePin, type OriginPoint } from "@/components/WorldMap.vue";

const { t } = useI18n();
const router = useRouter();
const auth = useAuthStore();
const theme = useThemeStore();
const connection = useConnectionStore();
const planStore = usePlanStore();
const message = useMessage();
const dialog = useDialog();

const loading = ref(true);
const connectBusy = ref(false);
const refreshingGroup = ref<string | null>(null);
const selecting = ref<string | null>(null);
const geoTesting = ref<string | null>(null);
// Per-node latency cache. Refreshed lazily on group expand or via the
// "test all" button. -1 sentinel means "tested but timed out".
const latency = reactive<Record<string, number>>({});
const nodeGeo = reactive<Record<string, NodeGeo>>({});

const health = ref<KernelHealth | null>(null);
const showLogModal = ref(false);
const logText = ref("");
const logLoading = ref(false);

// Critical: bundled mihomo missing — almost certainly a corrupted install.
const sidecarMissing = computed(
  () => health.value !== null && !health.value.mihomo_present,
);
// Soft hint. Meaning depends on platform:
//   - macOS: helper LaunchDaemon hasn't been installed yet (first connect
//     will trigger an admin auth prompt)
//   - Linux: bundled mihomo lacks cap_net_admin (AppImage / dev build) —
//     TUN will fall back to system-proxy until the user installs deb/rpm
const helperMissing = computed(() => health.value?.helper_present === false);
const hostPlatform = ref<string>("");
const helperMissingTitle = computed(() => {
  if (hostPlatform.value === "linux") return t("connect.health.helperMissingLinux");
  if (hostPlatform.value === "windows") return t("connect.health.helperMissingWindows");
  return t("connect.health.helperMissing");
});
const helperMissingBody = computed(() => {
  if (hostPlatform.value === "linux") return t("connect.health.helperMissingBodyLinux");
  if (hostPlatform.value === "windows") return t("connect.health.helperMissingBodyWindows");
  return t("connect.health.helperMissingBody");
});

const modeOptions = computed<Array<{ label: string; value: TunnelMode }>>(() => [
  { label: t("connect.mode.tun"), value: "tun" satisfies TunnelMode },
  { label: t("connect.mode.system_proxy"), value: "system_proxy" satisfies TunnelMode },
]);

const statusLabel = computed(() => {
  const s = connection.state;
  switch (s.kind) {
    case "disconnected":
      return t("connect.status.disconnected");
    case "connected":
      return t("connect.status.connected");
    case "error":
      return t("connect.status.error", { message: s.message });
    case "connecting":
      return t(`connect.status.connecting.${s.stage satisfies ConnectStage}`);
  }
});

const connectionPillType = computed<"success" | "warning" | "error" | "default">(() => {
  const k = connection.state.kind;
  if (k === "connected") return "success";
  if (k === "connecting") return "warning";
  if (k === "error") return "error";
  return "default";
});

// Selectable groups only — Direct/Reject/etc. show nothing useful for the user.
const selectableGroups = computed<ProxyGroup[]>(() =>
  connection.proxies.filter(
    (g) => g.type === "Selector" || g.type === "URLTest" || g.type === "Fallback",
  ),
);

type GeoPoint = {
  label: string;
  country: string;
  flag: string;
  lat: number;
  lon: number;
};

const worldMapRef = ref<InstanceType<typeof WorldMap> | null>(null);

const GEO_ALIASES: Array<GeoPoint & { keys: string[] }> = [
  { label: "Taipei", country: "台湾", flag: "🇹🇼", lat: 25.033, lon: 121.565, keys: ["台北", "taipei"] },
  { label: "Taiwan", country: "台湾", flag: "🇹🇼", lat: 23.697, lon: 120.961, keys: ["台湾", "台灣", "taiwan", " tw"] },
  { label: "Tokyo", country: "日本", flag: "🇯🇵", lat: 35.676, lon: 139.65, keys: ["东京", "東京", "tokyo"] },
  { label: "Osaka", country: "日本", flag: "🇯🇵", lat: 34.694, lon: 135.502, keys: ["大阪", "osaka"] },
  { label: "Japan", country: "日本", flag: "🇯🇵", lat: 36.204, lon: 138.253, keys: ["日本", "japan", " jp"] },
  { label: "Hong Kong", country: "香港", flag: "🇭🇰", lat: 22.319, lon: 114.169, keys: ["香港", "hong kong", "hongkong", " hk"] },
  { label: "Singapore", country: "新加坡", flag: "🇸🇬", lat: 1.352, lon: 103.82, keys: ["新加坡", "singapore", " sg"] },
  { label: "Seoul", country: "韩国", flag: "🇰🇷", lat: 37.566, lon: 126.978, keys: ["首尔", "首爾", "seoul"] },
  { label: "Korea", country: "韩国", flag: "🇰🇷", lat: 36.5, lon: 127.8, keys: ["韩国", "韓國", "korea", " kr"] },
  { label: "Los Angeles", country: "美国", flag: "🇺🇸", lat: 34.052, lon: -118.244, keys: ["洛杉矶", "洛杉磯", "los angeles", "la-"] },
  { label: "San Jose", country: "美国", flag: "🇺🇸", lat: 37.338, lon: -121.886, keys: ["圣何塞", "聖何塞", "san jose", "sanjose"] },
  { label: "New York", country: "美国", flag: "🇺🇸", lat: 40.713, lon: -74.006, keys: ["纽约", "紐約", "new york"] },
  { label: "United States", country: "美国", flag: "🇺🇸", lat: 39.828, lon: -98.579, keys: ["美国", "美國", "united states", "usa", " us"] },
  { label: "London", country: "英国", flag: "🇬🇧", lat: 51.507, lon: -0.128, keys: ["伦敦", "倫敦", "london"] },
  { label: "United Kingdom", country: "英国", flag: "🇬🇧", lat: 54.0, lon: -2.0, keys: ["英国", "英國", "united kingdom", " uk"] },
  { label: "Frankfurt", country: "德国", flag: "🇩🇪", lat: 50.11, lon: 8.682, keys: ["法兰克福", "法蘭克福", "frankfurt"] },
  { label: "Germany", country: "德国", flag: "🇩🇪", lat: 51.165, lon: 10.452, keys: ["德国", "德國", "germany", " de"] },
  { label: "Paris", country: "法国", flag: "🇫🇷", lat: 48.857, lon: 2.352, keys: ["巴黎", "paris", "法国", "法國", "france"] },
  { label: "Amsterdam", country: "荷兰", flag: "🇳🇱", lat: 52.367, lon: 4.904, keys: ["阿姆斯特丹", "amsterdam", "荷兰", "荷蘭", "netherlands", "nl"] },
  { label: "Sydney", country: "澳大利亚", flag: "🇦🇺", lat: -33.869, lon: 151.209, keys: ["悉尼", "sydney"] },
  { label: "Australia", country: "澳大利亚", flag: "🇦🇺", lat: -25.274, lon: 133.775, keys: ["澳大利亚", "澳洲", "australia", " au"] },
  { label: "Toronto", country: "加拿大", flag: "🇨🇦", lat: 43.653, lon: -79.383, keys: ["多伦多", "多倫多", "toronto"] },
  { label: "Canada", country: "加拿大", flag: "🇨🇦", lat: 56.13, lon: -106.347, keys: ["加拿大", "canada", " ca"] },
  { label: "Bangkok", country: "泰国", flag: "🇹🇭", lat: 13.756, lon: 100.501, keys: ["曼谷", "bangkok", "泰国", "泰國", "thailand"] },
  { label: "Ho Chi Minh City", country: "越南", flag: "🇻🇳", lat: 10.823, lon: 106.63, keys: ["胡志明", "越南", "vietnam", "vn"] },
  { label: "Manila", country: "菲律宾", flag: "🇵🇭", lat: 14.599, lon: 120.984, keys: ["马尼拉", "馬尼拉", "菲律宾", "菲律賓", "philippines"] },
  { label: "Kuala Lumpur", country: "马来西亚", flag: "🇲🇾", lat: 3.139, lon: 101.687, keys: ["吉隆坡", "马来", "馬來", "malaysia"] },
  { label: "Jakarta", country: "印尼", flag: "🇮🇩", lat: -6.208, lon: 106.846, keys: ["雅加达", "雅加達", "印尼", "indonesia"] },
  { label: "Mumbai", country: "印度", flag: "🇮🇳", lat: 19.076, lon: 72.878, keys: ["孟买", "孟買", "mumbai"] },
  { label: "India", country: "印度", flag: "🇮🇳", lat: 20.594, lon: 78.963, keys: ["印度", "india"] },
  { label: "Dubai", country: "阿联酋", flag: "🇦🇪", lat: 25.205, lon: 55.271, keys: ["迪拜", "dubai", "阿联酋", "阿聯酋", "uae"] },
  { label: "Istanbul", country: "土耳其", flag: "🇹🇷", lat: 41.008, lon: 28.978, keys: ["伊斯坦布尔", "土耳其", "turkey", "istanbul"] },
  { label: "Moscow", country: "俄罗斯", flag: "🇷🇺", lat: 55.756, lon: 37.617, keys: ["莫斯科", "俄罗斯", "俄羅斯", "russia"] },
  { label: "Shanghai", country: "中国", flag: "🇨🇳", lat: 31.231, lon: 121.474, keys: ["上海", "shanghai"] },
  { label: "Beijing", country: "中国", flag: "🇨🇳", lat: 39.904, lon: 116.407, keys: ["北京", "beijing", "中国", "中國", "china"] },
  { label: "Macau", country: "澳门", flag: "🇲🇴", lat: 22.199, lon: 113.544, keys: ["澳门", "澳門", "macau", "macao"] },
];

function normalizeNodeName(name: string): string {
  return ` ${name
    .toLowerCase()
    .replace(/[|｜_\-·•/()[\]{}]+/g, " ")
    .replace(/\s+/g, " ")} `;
}

function locateNode(name: string): GeoPoint | null {
  const normalized = normalizeNodeName(name);
  const hit = GEO_ALIASES.find((entry) => entry.keys.some((key) => normalized.includes(key.toLowerCase())));
  if (!hit) return null;
  const { keys: _keys, ...point } = hit;
  return point;
}

const nodeLocationByName = computed<Record<string, GeoPoint | null>>(() => {
  const out: Record<string, GeoPoint | null> = {};
  for (const group of selectableGroups.value) {
    for (const node of group.all) out[node] = locateNode(node);
  }
  if (connection.effectiveProxy) out[connection.effectiveProxy] = locateNode(connection.effectiveProxy);
  return out;
});

function geoToPoint(geo: NodeGeo): GeoPoint {
  return {
    label: [geo.city, geo.country].filter(Boolean).join(", ") || geo.ip,
    country: geo.country || geo.ip,
    flag: "●",
    lat: geo.lat,
    lon: geo.lon,
  };
}

function locationForNode(name: string): GeoPoint | null {
  return nodeGeo[name] ? geoToPoint(nodeGeo[name]) : (nodeLocationByName.value[name] ?? locateNode(name));
}

const selectedLocation = computed(() =>
  connection.currentProxy
    ? locationForNode(connection.currentProxy)
    : connection.effectiveProxy
      ? locationForNode(connection.effectiveProxy)
      : null,
);

// User origin pin — derived from browser timezone since we don't ask for
// geolocation. Falls back to null (no pin rendered) for timezones we don't
// have a coarse city anchor for.
const originPin = computed<OriginPoint | null>(() => {
  const timezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
  switch (timezone) {
    case "Asia/Shanghai":
      return { lat: 31.231, lon: 121.474, label: "You" };
    case "Asia/Hong_Kong":
      return { lat: 22.319, lon: 114.169, label: "You" };
    case "Asia/Taipei":
      return { lat: 25.033, lon: 121.565, label: "You" };
    case "Asia/Tokyo":
      return { lat: 35.676, lon: 139.65, label: "You" };
    case "Asia/Singapore":
      return { lat: 1.352, lon: 103.82, label: "You" };
    case "Asia/Seoul":
      return { lat: 37.566, lon: 126.978, label: "You" };
    case "America/Los_Angeles":
      return { lat: 34.052, lon: -118.244, label: "You" };
    case "America/New_York":
      return { lat: 40.713, lon: -74.006, label: "You" };
    case "Europe/London":
      return { lat: 51.507, lon: -0.128, label: "You" };
    case "Europe/Berlin":
      return { lat: 52.52, lon: 13.405, label: "You" };
    default:
      return null;
  }
});

const mapPins = computed<NodePin[]>(() => {
  const byLocation = new globalThis.Map<string, NodePin>();
  const activeNode = connection.effectiveProxy;
  const activeSelector = connection.currentProxy;
  const names = selectableGroups.value.flatMap((g) => g.all);
  if (activeSelector) names.unshift(activeSelector);
  if (activeNode) names.unshift(activeNode);

  for (const node of names) {
    const loc = locationForNode(node);
    if (!loc) continue;
    const key = `${loc.lat.toFixed(3)},${loc.lon.toFixed(3)}`;
    const isActive = node === activeNode || node === activeSelector;
    const current = byLocation.get(key);
    if (current) {
      current.count += 1;
      current.active = current.active || isActive;
      continue;
    }
    const geo = nodeGeo[node];
    byLocation.set(key, {
      id: key,
      lat: loc.lat,
      lon: loc.lon,
      label: loc.label,
      country: loc.country,
      ip: geo?.ip,
      count: 1,
      active: isActive,
    });
  }

  return [...byLocation.values()]
    .sort((a, b) => Number(b.active) - Number(a.active) || b.count - a.count)
    .slice(0, 40);
});

function findNodeAtLocation(lat: number, lon: number): { group: string; node: string } | null {
  const key = `${lat.toFixed(3)},${lon.toFixed(3)}`;
  for (const g of selectableGroups.value) {
    for (const node of g.all) {
      const loc = locationForNode(node);
      if (!loc) continue;
      if (`${loc.lat.toFixed(3)},${loc.lon.toFixed(3)}` === key) {
        return { group: g.name, node };
      }
    }
  }
  return null;
}

function onMapPinClick(id: string) {
  const pin = mapPins.value.find((p) => p.id === id);
  if (!pin) return;
  worldMapRef.value?.flyTo(pin.lat, pin.lon, 3.2);
  // Single-node cluster → also switch to it. Multi-node → leave selection to the sidebar list.
  if (pin.count === 1) {
    const target = findNodeAtLocation(pin.lat, pin.lon);
    if (target) void selectNode(target.group, target.node);
  }
}

function nodeFlag(name: string): string {
  if (nodeGeo[name]) return "●";
  return nodeLocationByName.value[name]?.flag ?? "▾";
}

async function toggleConnection(value: boolean) {
  if (connectBusy.value) return;
  connectBusy.value = true;
  try {
    if (value) {
      await connection.connect();
    } else {
      await connection.disconnect();
    }
  } catch (e) {
    // KernelManager auto-downgrades TUN→system_proxy on the consent /
    // service-missing / not-permitted paths, so those LauncherError variants
    // never surface here. Anything that reaches this catch is a real failure
    // (kernel start timeout, IPC error, …) — show it raw.
    message.error(formatError(e, t));
  } finally {
    connectBusy.value = false;
  }
}

// One-shot toast when the kernel quietly downgraded the user's TUN request
// to system_proxy. The watch guards on a false→true transition so toggling
// the switch off and back on doesn't re-toast unless the downgrade reoccurs.
watch(
  () => connection.wasDowngraded,
  (now, prev) => {
    if (now && !prev) {
      message.warning(t("connect.downgraded"));
    }
  },
);

watch(
  () => [connection.isConnected, connection.primaryGroup?.name, connection.currentProxy] as const,
  ([connected, group, node]) => {
    if (!connected || !group || !node || nodeGeo[node] || geoTesting.value) return;
    void testNodeGeo(group, node, true);
  },
);

// One-shot subscribe-wide geo refresh. Triggered the first time we have a
// non-empty selectable group list — works whether or not the user is currently
// connected because the lookup reads the live config.yaml (written by core on
// every connect) and queries ip-api.com directly, not through the tunnel.
let didBatchGeo = false;
async function refreshGeoBatch() {
  try {
    const map = await api.resolveNodeGeoBatch();
    for (const [name, geo] of Object.entries(map)) {
      nodeGeo[name] = geo;
    }
    if (Object.keys(map).length > 0) await connection.refreshProxies();
  } catch (e) {
    // Best-effort — the map falls back to name-based aliases when geo is missing.
    console.warn("resolveNodeGeoBatch failed", e);
  }
}
watch(
  () => selectableGroups.value.length,
  (n) => {
    if (n > 0 && !didBatchGeo) {
      didBatchGeo = true;
      void refreshGeoBatch();
    }
  },
);

async function onModeChange(next: TunnelMode) {
  // setMode auto-reconnects when currently connected (so the new mode actually
  // takes effect). Surface that so the user understands why the connection
  // pill briefly drops to "connecting".
  const reconnecting = connection.isConnected && next !== connection.currentMode;
  try {
    if (reconnecting) {
      message.info(t("connect.modeReconnect"));
    }
    await connection.setMode(next);
  } catch (e) {
    message.error(formatError(e, t));
  }
}

async function refreshHealth() {
  try {
    health.value = await api.kernelHealth();
  } catch (e) {
    // Banner is purely advisory — silently swallow.
    console.warn("kernel_health failed", e);
  }
}

async function openLogModal() {
  showLogModal.value = true;
  await refreshLog();
}

async function refreshLog() {
  logLoading.value = true;
  try {
    logText.value = await api.tailKernelLog();
  } catch (e) {
    logText.value = formatError(e, t);
  } finally {
    logLoading.value = false;
  }
}

async function selectNode(group: string, name: string) {
  if (selecting.value) return;
  selecting.value = `${group}::${name}`;
  try {
    await connection.selectProxy(group, name);
  } catch (e) {
    message.error(formatError(e, t));
  } finally {
    selecting.value = null;
  }
}

async function testGroupLatency(group: ProxyGroup) {
  if (refreshingGroup.value) return;
  refreshingGroup.value = group.name;
  // Run probes concurrently but cap to 8 in flight so we don't slam the kernel.
  const queue = [...group.all];
  const inflight: Promise<void>[] = [];
  const runOne = async (name: string) => {
    try {
      const ms = await api.latencyTest(name);
      latency[name] = ms;
    } catch {
      latency[name] = -1;
    }
  };
  while (queue.length > 0 || inflight.length > 0) {
    while (inflight.length < 8 && queue.length > 0) {
      const name = queue.shift()!;
      const p = runOne(name).finally(() => {
        const idx = inflight.indexOf(p);
        if (idx >= 0) inflight.splice(idx, 1);
      });
      inflight.push(p);
    }
    if (inflight.length > 0) await Promise.race(inflight);
  }
  refreshingGroup.value = null;
}

async function testNodeGeo(group: string, name: string, silent = false) {
  if (!connection.isConnected) {
    if (!silent) message.warning("请先连接，连接成功后才能通过节点出口 IP 定位");
    return;
  }
  if (geoTesting.value) return;
  geoTesting.value = `${group}::${name}`;
  try {
    const geo = await api.nodeGeoTest(group, name);
    nodeGeo[name] = geo;
    await connection.refreshProxies();
    if (!silent) message.success(`${name} 定位到 ${geo.city || geo.country || geo.ip}`);
  } catch (e) {
    if (!silent) message.error(formatError(e, t));
  } finally {
    geoTesting.value = null;
  }
}

async function testVisibleGeo(group: ProxyGroup) {
  for (const member of group.all.slice(0, 8)) {
    if (nodeGeo[member]) continue;
    await testNodeGeo(group.name, member, true);
  }
}

function latencyText(name: string): string {
  const v = latency[name];
  if (v === undefined) return "—";
  if (v < 0 || v >= 5000) return t("connect.nodes.timeout");
  return `${v} ms`;
}

function latencyTone(name: string): "success" | "warning" | "error" | "default" {
  const v = latency[name];
  if (v === undefined) return "default";
  if (v < 0) return "error";
  if (v < 200) return "success";
  if (v < 600) return "warning";
  return "error";
}

function fmtSpeed(bytesPerSec: number): string {
  if (!bytesPerSec) return "0 B/s";
  const units = ["B/s", "KB/s", "MB/s", "GB/s"];
  let v = bytesPerSec;
  let i = 0;
  while (v >= 1024 && i < units.length - 1) {
    v /= 1024;
    i++;
  }
  return `${v.toFixed(1)} ${units[i]}`;
}

async function refresh() {
  loading.value = true;
  try {
    await Promise.all([
      auth.refreshUser(),
      auth.refreshSubscribe(),
      connection.hydrate(),
      // Best-effort plan catalog warm-up so the cards can show plan names
      // instead of bare numeric ids. Failures are non-fatal — the UI falls
      // back to `#<id>` automatically.
      planStore.ensure().catch(() => {}),
    ]);
  } catch (e) {
    message.error(formatError(e, t));
    if (
      typeof e === "object" &&
      e !== null &&
      "kind" in e &&
      (e as { kind: string }).kind === "unauthorized"
    ) {
      await auth.logout();
      router.push({ name: "login" });
    }
  } finally {
    loading.value = false;
  }
}

async function autoConnectAfterLogin() {
  if (sessionStorage.getItem(AUTO_CONNECT_AFTER_LOGIN_KEY) !== "1") return;
  sessionStorage.removeItem(AUTO_CONNECT_AFTER_LOGIN_KEY);
  if (connection.isConnected || connection.isBusy) return;

  connectBusy.value = true;
  try {
    message.info(t("connect.autoConnecting"));
    await connection.connect();
    message.success(t("connect.autoConnected"));
  } catch (e) {
    message.error(formatError(e, t));
  } finally {
    connectBusy.value = false;
  }
}

async function onLogout() {
  if (connection.isConnected) {
    try {
      await connection.disconnect();
    } catch {
      // best-effort
    }
  }
  await auth.logout();
  router.push({ name: "login" });
}

// Header account menu — keeps the inline buttons short while adding room for
// future user-center entries (tickets, invitations, …) without re-cluttering
// the bar.
const accountMenu = computed<DropdownOption[]>(() => [
  { label: t("home.menu.plans"), key: "plans" },
  { label: t("home.menu.orders"), key: "orders" },
  { label: t("home.menu.tickets"), key: "tickets" },
  { label: t("home.menu.notices"), key: "notices" },
  { type: "divider", key: "d1" },
  { label: t("home.menu.helper"), key: "helper" },
  { label: t("home.menu.kernelInfo"), key: "kernel_info" },
  { label: t("home.menu.checkUpdate"), key: "check_update" },
  { type: "divider", key: "d2" },
  { label: t("home.logout"), key: "logout", props: { style: "color: var(--n-error-color, #d03050)" } },
]);

function onAccountSelect(key: string) {
  switch (key) {
    case "plans":
      router.push({ name: "plans" });
      break;
    case "orders":
      router.push({ name: "orders" });
      break;
    case "tickets":
      router.push({ name: "tickets" });
      break;
    case "notices":
      router.push({ name: "notices" });
      break;
    case "helper":
      void openHelperPanel();
      break;
    case "kernel_info":
      void openKernelInfoPanel();
      break;
    case "check_update":
      void onCheckUpdate();
      break;
    case "logout":
      void onLogout();
      break;
  }
}

// Manual update entry. tauri-plugin-updater fetches `latest.json` from the
// configured endpoint, verifies the bundle's ed25519 signature, and only
// then surfaces an `Update` object — so by the time we render the dialog
// the artifact has already been authenticated. We keep the install path
// behind an explicit user confirm because `relaunch()` quits the app.
const checkingUpdate = ref(false);
async function onCheckUpdate() {
  if (checkingUpdate.value) return;
  checkingUpdate.value = true;
  try {
    const update = await check();
    if (!update) {
      message.success(t("updater.upToDate"));
      return;
    }
    dialog.warning({
      title: t("updater.availableTitle", { version: update.version }),
      content: update.body || t("updater.availableBody"),
      positiveText: t("updater.installNow"),
      negativeText: t("updater.later"),
      onPositiveClick: async () => {
        try {
          await update.downloadAndInstall();
          await relaunch();
        } catch (e) {
          message.error(formatError(e, t));
        }
      },
    });
  } catch (e) {
    message.error(formatError(e, t));
  } finally {
    checkingUpdate.value = false;
  }
}

// Helper-service management (macOS only). The backend hides the panel by
// reporting `supported: false` on Linux/Windows; the menu entry is still
// surfaced uniformly so users on those platforms see why it's empty.
const showHelperModal = ref(false);
const helperStatus = ref<HelperStatus | null>(null);
const helperBusy = ref(false);
async function loadHelperStatus() {
  try {
    helperStatus.value = await api.helperStatus();
  } catch (e) {
    message.error(formatError(e, t));
  }
}
async function openHelperPanel() {
  showHelperModal.value = true;
  await loadHelperStatus();
}
async function onHelperInstall() {
  if (helperBusy.value) return;
  helperBusy.value = true;
  try {
    await api.helperInstall();
    message.success(t("helper.installed"));
    await loadHelperStatus();
    void refreshHealth();
  } catch (e) {
    message.error(formatError(e, t));
  } finally {
    helperBusy.value = false;
  }
}
async function onHelperUninstall() {
  if (helperBusy.value) return;
  dialog.warning({
    title: t("helper.confirmUninstallTitle"),
    content: t("helper.confirmUninstallBody"),
    positiveText: t("helper.uninstallNow"),
    negativeText: t("helper.cancel"),
    onPositiveClick: async () => {
      helperBusy.value = true;
      try {
        if (connection.isConnected) {
          await connection.disconnect();
        }
        await api.helperUninstall();
        message.success(t("helper.uninstalled"));
        await loadHelperStatus();
        void refreshHealth();
      } catch (e) {
        message.error(formatError(e, t));
      } finally {
        helperBusy.value = false;
      }
    },
  });
}

// Kernel info modal — shows the bundled mihomo version + path. Auto-update
// of the kernel rides with the app updater, so we deliberately do not
// expose a "check for kernel update" button here.
const showKernelInfoModal = ref(false);
const kernelVersion = ref<KernelVersion | null>(null);
const kernelInfoLoading = ref(false);
async function openKernelInfoPanel() {
  showKernelInfoModal.value = true;
  if (kernelVersion.value !== null) return;
  kernelInfoLoading.value = true;
  try {
    kernelVersion.value = await api.kernelVersion();
  } catch (e) {
    message.error(formatError(e, t));
  } finally {
    kernelInfoLoading.value = false;
  }
}

async function onCopySubscribe() {
  if (!auth.subscribe) return;
  await navigator.clipboard.writeText(auth.subscribe.subscribe_url);
  message.success(t("home.copied"));
}

// Backend emits this the first time it intercepts the main window's close
// (closing now hides to tray). The localStorage flag keeps the toast a
// one-shot — repeating it every close would just be noise.
const TRAY_HINT_KEY = "xboard.trayHintShown";
const AUTO_CONNECT_AFTER_LOGIN_KEY = "xboard.autoConnectAfterLogin";
let unlistenTrayHint: UnlistenFn | null = null;

onMounted(async () => {
  await refresh();
  await autoConnectAfterLogin();
  void refreshHealth();
  // Best-effort: pick the helper-missing copy variant. Failure here just
  // keeps the default (mac) wording, which is fine — the alert only shows
  // when helper_present is false anyway, which is platform-correlated.
  try {
    hostPlatform.value = await platform();
  } catch {
    hostPlatform.value = "";
  }
  unlistenTrayHint = await listen("xboard://hidden-to-tray", () => {
    if (localStorage.getItem(TRAY_HINT_KEY) === "1") return;
    message.info(t("connect.hiddenToTray"), { duration: 5000 });
    localStorage.setItem(TRAY_HINT_KEY, "1");
  });
});
onBeforeUnmount(() => {
  void connection.dispose();
  if (unlistenTrayHint) {
    unlistenTrayHint();
    unlistenTrayHint = null;
  }
});

const yuan = (cents: number) => (cents / 100).toFixed(2);
const trafficUsed = computed(() => {
  const s = auth.subscribe;
  if (!s) return 0;
  return s.u + s.d;
});
const trafficPct = computed(() => {
  const s = auth.subscribe;
  if (!s || !s.transfer_enable) return 0;
  return Math.min(100, (trafficUsed.value / s.transfer_enable) * 100);
});

function fmtBytes(bytes: number): string {
  if (!bytes) return "0 B";
  const units = ["B", "KB", "MB", "GB", "TB"];
  let v = bytes;
  let i = 0;
  while (v >= 1024 && i < units.length - 1) {
    v /= 1024;
    i++;
  }
  return `${v.toFixed(2)} ${units[i]}`;
}

function fmtExpiry(ts: number | null | undefined): string {
  if (!ts) return t("home.expiryNever");
  return new Date(ts * 1000).toLocaleString();
}

// Friendly plan name for the user info card. Falls back to `#<id>` (or
// "—" when the user has no plan yet) so a slow / failed plan-catalog
// fetch never leaves the card looking broken.
const planLabel = computed(() => {
  const id = auth.userInfo?.plan_id ?? null;
  if (id == null) return "—";
  return planStore.nameFor(id) ?? `#${id}`;
});

// Subscription state hint — drives a small CTA panel that links the user
// to the right page (Plans for new/expiring, Plans for upgrade when
// traffic-bound). Intentionally tier-based instead of a bunch of booleans
// so the template only has to switch on one value.
type SubState = "none" | "expired" | "expiringSoon" | "trafficLow" | "ok";
const SEVEN_DAYS_S = 7 * 24 * 60 * 60;
const subState = computed<SubState>(() => {
  const s = auth.subscribe;
  if (!s) return "none";
  const nowS = Math.floor(Date.now() / 1000);
  if (s.expired_at !== null && s.expired_at !== undefined) {
    if (s.expired_at <= nowS) return "expired";
    if (s.expired_at - nowS <= SEVEN_DAYS_S) return "expiringSoon";
  }
  if (s.transfer_enable > 0 && trafficUsed.value / s.transfer_enable >= 0.95) {
    return "trafficLow";
  }
  return "ok";
});

const subStateMeta = computed<{
  type: "default" | "info" | "success" | "warning" | "error";
  title: string;
  body: string;
  ctaLabel: string;
} | null>(() => {
  switch (subState.value) {
    case "none":
      return {
        type: "info",
        title: t("home.cue.noneTitle"),
        body: t("home.cue.noneBody"),
        ctaLabel: t("home.cue.cta.browse"),
      };
    case "expired":
      return {
        type: "error",
        title: t("home.cue.expiredTitle"),
        body: t("home.cue.expiredBody"),
        ctaLabel: t("home.cue.cta.renew"),
      };
    case "expiringSoon":
      return {
        type: "warning",
        title: t("home.cue.expiringTitle"),
        body: t("home.cue.expiringBody", {
          date: fmtExpiry(auth.subscribe?.expired_at),
        }),
        ctaLabel: t("home.cue.cta.renew"),
      };
    case "trafficLow":
      return {
        type: "warning",
        title: t("home.cue.trafficLowTitle"),
        body: t("home.cue.trafficLowBody"),
        ctaLabel: t("home.cue.cta.upgrade"),
      };
    default:
      return null;
  }
});

function onCueAction() {
  router.push({ name: "plans" });
}

</script>

<template>
  <NLayout class="home-shell">
    <NLayoutHeader bordered class="home-header">
      <div class="brand">
        <span class="brand-mark" />
        <NText strong class="brand-text">{{ t("app.title") }}</NText>
        <NTag :type="connectionPillType" size="small" round class="brand-pill">
          <span class="status-dot" :class="{ pulse: connection.isBusy }" />
          {{ statusLabel }}
        </NTag>
      </div>
      <NSpace :size="8" align="center">
        <NButton size="small" quaternary @click="theme.toggle">
          {{ theme.dark ? "☀︎" : "☾" }}
        </NButton>
        <NButton size="small" quaternary :loading="loading" @click="refresh">
          {{ t("home.refresh") }}
        </NButton>
        <NDropdown
          trigger="click"
          :options="accountMenu"
          :show-arrow="true"
          placement="bottom-end"
          @select="onAccountSelect"
        >
          <NButton size="small" quaternary class="account-btn">
            <span class="header-email">{{ auth.session?.email ?? "" }}</span>
            <span class="caret">▾</span>
          </NButton>
        </NDropdown>
      </NSpace>
    </NLayoutHeader>

    <NLayoutContent class="home-content">
      <div class="proton-workspace">
        <NAlert
          v-if="sidecarMissing"
          type="error"
          :show-icon="true"
          :title="t('connect.health.binaryMissing')"
          class="health-alert"
        >
          {{
            t("connect.health.binaryMissingBody", {
              path: health?.mihomo_path ?? "",
            })
          }}
        </NAlert>
        <NAlert
          v-else-if="helperMissing"
          :type="hostPlatform === 'linux' ? 'warning' : 'info'"
          :show-icon="true"
          :title="helperMissingTitle"
          class="health-alert"
        >
          {{ helperMissingBody }}
        </NAlert>

        <aside class="proton-sidebar">
          <div class="protection-block">
            <div class="protection-state" :class="{ connected: connection.isConnected }">
              {{ connection.isConnected ? t("connect.status.connected") : "未受保护" }}
            </div>
            <div class="ip-line">IP · {{ connection.isConnected ? "127.0.0.1" : "—" }}</div>
            <button
              class="quick-connect"
              type="button"
              :disabled="connection.isBusy || connectBusy"
              @click="toggleConnection(!connection.isConnected)"
            >
              {{ connection.isConnected ? t("connect.button.connected") : "快速连接" }}
            </button>
          </div>

          <div class="sidebar-tabs">
            <button class="is-active" type="button">国家</button>
            <button type="button">收藏</button>
          </div>

          <div class="search-box">⌕ 搜索国家或节点</div>

          <div class="mode-strip">
            <button
              v-for="opt in modeOptions"
              :key="opt.value"
              type="button"
              :class="{ active: connection.currentMode === opt.value }"
              @click="onModeChange(opt.value)"
            >
              {{ opt.label }}
            </button>
          </div>

          <div class="node-section">
            <div class="section-title">
              <span>免费连接（{{ selectableGroups.length || 1 }}）</span>
              <button type="button" @click="connection.refreshProxies()">↻</button>
            </div>
            <button
              class="node-row fastest"
              type="button"
              :disabled="connection.isBusy || connectBusy"
              @click="toggleConnection(true)"
            >
              <span class="flag">⚡</span>
              <span>最快服务器</span>
              <span class="latency">52ms</span>
            </button>

            <NScrollbar style="max-height: 420px">
              <div v-if="selectableGroups.length === 0" class="empty-nodes">
                {{ connection.isConnected ? t("connect.nodes.loading") : t("connect.nodes.needsConnect") }}
              </div>
              <template v-for="g in selectableGroups" :key="g.name">
	                <div class="group-title">
	                  <span>{{ g.name }}</span>
	                  <span class="group-actions">
	                    <button
	                      type="button"
	                      :disabled="!!geoTesting || !connection.isConnected"
	                      @click="testVisibleGeo(g)"
	                    >
	                      IP定位
	                    </button>
	                    <button
	                      type="button"
	                      :disabled="refreshingGroup === g.name"
	                      @click="testGroupLatency(g)"
	                    >
	                      {{ t("connect.nodes.testAll") }}
	                    </button>
	                  </span>
	                </div>
	                <button
                  v-for="member in g.all.slice(0, 8)"
                  :key="`${g.name}-${member}`"
                  type="button"
                  class="node-row"
                  :class="{ selected: member === g.now }"
                  :disabled="!!selecting"
                  @click="selectNode(g.name, member)"
	                >
	                  <span class="flag">{{ nodeFlag(member) }}</span>
	                  <span class="node-label">{{ member }}</span>
	                  <span class="node-meta">
	                    <small v-if="nodeGeo[member]">{{ nodeGeo[member].city || nodeGeo[member].country }}</small>
	                    <small v-else :class="`tone-${latencyTone(member)}`">{{ latencyText(member) }}</small>
	                    <small
	                      class="geo-action"
	                      :class="{ loading: geoTesting === `${g.name}::${member}` }"
	                      @click.stop="testNodeGeo(g.name, member)"
	                    >
	                      {{ nodeGeo[member] ? nodeGeo[member].ip : "定位" }}
	                    </small>
	                  </span>
	                </button>
              </template>
            </NScrollbar>
          </div>
        </aside>

        <main class="map-stage">
          <WorldMap
            ref="worldMapRef"
            class="map-canvas"
            :pins="mapPins"
            :origin="originPin"
            @pin-click="onMapPinClick"
          />
          <div class="map-shade" aria-hidden="true" />
          <div class="status-crown">
            <strong :class="{ connected: connection.isConnected }">
              {{ connection.isConnected ? "已保护" : "未受保护" }}
            </strong>
            <span>⌂</span>
          </div>
          <div class="server-hero">
            <div class="server-badge">{{ connection.isConnected ? "✓" : "!" }}</div>
            <h1>{{ connection.effectiveProxy ?? "最快服务器" }}</h1>
            <p v-if="connection.currentProxy && connection.effectiveProxy !== connection.currentProxy">
              {{ connection.primaryGroup?.name }} / {{ connection.currentProxy }}
            </p>
            <p v-else-if="selectedLocation">
              {{ selectedLocation.country }} · {{ selectedLocation.label }}
            </p>
            <p v-else>自动选择最优节点</p>
            <button
              class="connect-pill"
              type="button"
              :disabled="connection.isBusy || connectBusy"
              @click="toggleConnection(!connection.isConnected)"
            >
              <span>⏻</span>
              {{ connection.isConnected ? "断开连接" : "连接" }}
            </button>
          </div>

          <div class="bottom-status">
            <strong :class="{ danger: !connection.isConnected }">
              {{ connection.isConnected ? "受保护" : "未保护" }}
            </strong>
            <span>{{ statusLabel }}</span>
            <span v-if="connection.isConnected">
              ↑ {{ fmtSpeed(connection.traffic.up) }} · ↓ {{ fmtSpeed(connection.traffic.down) }}
            </span>
            <span class="map-attribution">© OpenFreeMap © OpenStreetMap</span>
          </div>
        </main>

        <aside class="right-rail">
          <button type="button" @click="refresh">↻<span>{{ t("home.refresh") }}</span></button>
          <button type="button" @click="openLogModal">☰<span>{{ t("connect.viewLogs") }}</span></button>
          <button type="button" @click="router.push({ name: 'plans' })">◇<span>{{ t("home.menu.plans") }}</span></button>
          <button type="button" @click="router.push({ name: 'tickets' })">○<span>{{ t("home.menu.tickets") }}</span></button>
          <button type="button" class="rail-bottom" @click="onAccountSelect('helper')">⚙<span>{{ t("helper.title") }}</span></button>
        </aside>

        <section class="account-dock">
          <div class="dock-card">
            <span>{{ t("home.balance") }}</span>
            <strong>¥ {{ yuan(auth.userInfo?.balance ?? 0) }}</strong>
          </div>
          <div class="dock-card">
            <span>{{ t("home.commission") }}</span>
            <strong>¥ {{ yuan(auth.userInfo?.commission_balance ?? 0) }}</strong>
          </div>
          <div class="dock-card plan-card">
            <span>{{ t("home.plan") }}</span>
            <strong>{{ planLabel }}</strong>
          </div>
          <div class="dock-card wide">
            <span>{{ t("home.traffic") }}</span>
            <div class="traffic-bar"><i :style="{ width: `${trafficPct}%` }" /></div>
            <small>
              {{ fmtBytes(trafficUsed) }} / {{ fmtBytes(auth.subscribe?.transfer_enable ?? 0) }}
              · {{ fmtExpiry(auth.subscribe?.expired_at) }}
            </small>
          </div>
          <div v-if="auth.subscribe" class="dock-card subscribe-dock">
            <span>{{ t("home.subscribe") }}</span>
            <code>{{ auth.subscribe.subscribe_url }}</code>
            <button type="button" @click="onCopySubscribe">{{ t("home.copy") }}</button>
          </div>
          <NAlert
            v-if="subStateMeta"
            :type="subStateMeta.type"
            :title="subStateMeta.title"
            :show-icon="false"
            class="cue-alert"
          >
            <div class="cue-body">
              <span>{{ subStateMeta.body }}</span>
              <NButton size="small" type="primary" ghost @click="onCueAction">
                {{ subStateMeta.ctaLabel }}
              </NButton>
            </div>
          </NAlert>
        </section>
      </div>
    </NLayoutContent>

    <NModal
      v-model:show="showLogModal"
      preset="card"
      :title="t('connect.logModal.title')"
      style="max-width: 760px"
      :bordered="false"
      size="huge"
    >
      <template #header-extra>
        <NButton size="small" :loading="logLoading" @click="refreshLog">
          {{ t("connect.logModal.refresh") }}
        </NButton>
      </template>
      <NScrollbar style="max-height: 60vh">
        <pre v-if="logText" class="log-pre">{{ logText }}</pre>
        <NEmpty v-else :description="t('connect.logModal.empty')" />
      </NScrollbar>
      <NText v-if="health" depth="3" class="log-path">
        {{ health.work_dir }}/mihomo.log
      </NText>
    </NModal>

    <NModal
      v-model:show="showHelperModal"
      preset="card"
      :title="t('helper.title')"
      style="max-width: 560px"
      :bordered="false"
      size="huge"
    >
      <template #header-extra>
        <NButton size="small" @click="loadHelperStatus">
          {{ t("home.refresh") }}
        </NButton>
      </template>
      <NSpace v-if="!helperStatus" justify="center" style="padding: 24px 0">
        <NSpin />
      </NSpace>
      <div v-else-if="!helperStatus.supported">
        <NAlert type="info" :show-icon="false">
          {{ t("helper.unsupported") }}
        </NAlert>
      </div>
      <div v-else class="helper-panel">
        <NSpace align="center" :size="8">
          <NTag
            :type="helperStatus.installed ? 'success' : 'warning'"
            :bordered="false"
          >
            {{
              helperStatus.installed
                ? t("helper.tag.installed")
                : t("helper.tag.notInstalled")
            }}
          </NTag>
          <NTag
            v-if="helperStatus.installed"
            :type="helperStatus.reachable ? 'success' : 'error'"
            :bordered="false"
          >
            {{
              helperStatus.reachable
                ? t("helper.tag.reachable")
                : t("helper.tag.unreachable")
            }}
          </NTag>
        </NSpace>
        <NText depth="3" class="helper-tip">
          {{
            helperStatus.installed
              ? t("helper.bodyInstalled")
              : t("helper.bodyMissing")
          }}
        </NText>
        <div v-if="helperStatus.helper_path" class="helper-paths">
          <NText depth="3">{{ helperStatus.helper_path }}</NText>
          <NText v-if="helperStatus.plist_path" depth="3">
            {{ helperStatus.plist_path }}
          </NText>
        </div>
        <NSpace>
          <NButton
            type="primary"
            :loading="helperBusy"
            @click="onHelperInstall"
          >
            {{
              helperStatus.installed
                ? t("helper.reinstall")
                : t("helper.installNow")
            }}
          </NButton>
          <NButton
            v-if="helperStatus.installed"
            :loading="helperBusy"
            @click="onHelperUninstall"
          >
            {{ t("helper.uninstall") }}
          </NButton>
        </NSpace>
      </div>
    </NModal>

    <NModal
      v-model:show="showKernelInfoModal"
      preset="card"
      :title="t('kernelInfo.title')"
      style="max-width: 560px"
      :bordered="false"
      size="huge"
    >
      <NSpace v-if="kernelInfoLoading" justify="center" style="padding: 24px 0">
        <NSpin />
      </NSpace>
      <div v-else-if="kernelVersion" class="kernel-info">
        <NSpace align="center" :size="8">
          <NTag type="success" :bordered="false">
            {{ kernelVersion.version || t("kernelInfo.unknownVersion") }}
          </NTag>
        </NSpace>
        <NText depth="3" class="helper-tip">
          {{ t("kernelInfo.bundledHint") }}
        </NText>
        <pre class="log-pre">{{ kernelVersion.raw }}</pre>
        <NText depth="3" class="log-path">
          {{ kernelVersion.mihomo_path }}
        </NText>
      </div>
      <NEmpty v-else :description="t('kernelInfo.empty')" />
    </NModal>
  </NLayout>
</template>

<style scoped>
.home-shell {
  min-height: 100vh;
  color: #f8f7ff;
  background: #15121f;
}

.home-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 56px;
  padding: 0 24px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  background: #2b2735;
}

.brand,
.brand-pill,
.account-btn {
  display: inline-flex;
  align-items: center;
  gap: 10px;
}

.brand-mark {
  width: 26px;
  height: 26px;
  border-radius: 8px;
  background: linear-gradient(135deg, #8a5cf6, #00c48c);
}

.brand-text {
  color: #f8f7ff;
  font-size: 16px;
}

.brand-pill {
  color: #f8f7ff;
  background: rgba(255, 255, 255, 0.07);
  border-color: rgba(255, 255, 255, 0.12);
}

.status-dot {
  width: 7px;
  height: 7px;
  border-radius: 999px;
  background: currentColor;
}
.status-dot.pulse { animation: dot-pulse 1.4s ease-in-out infinite; }
@keyframes dot-pulse {
  0%, 100% { opacity: 0.45; transform: scale(0.85); }
  50% { opacity: 1; transform: scale(1.15); }
}

.header-email {
  max-width: 220px;
  overflow: hidden;
  color: #f8f7ff;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.home-content {
  min-height: calc(100vh - 56px);
}

.proton-workspace {
  display: grid;
  grid-template-columns: 330px minmax(520px, 1fr) 88px;
  grid-template-rows: minmax(0, 1fr) auto;
  min-height: calc(100vh - 56px);
  background:
    radial-gradient(circle at 48% 20%, rgba(98, 70, 180, 0.22), transparent 28%),
    linear-gradient(135deg, #171320 0%, #221c30 50%, #14111a 100%);
}

.health-alert {
  grid-column: 1 / -1;
  margin: 14px;
}

.proton-sidebar {
  grid-row: 1 / 3;
  min-height: 100%;
  padding: 26px 22px;
  background: rgba(18, 15, 25, 0.92);
  border-right: 1px solid rgba(255, 255, 255, 0.08);
}

.protection-block {
  padding-bottom: 26px;
}

.protection-state {
  color: #ff6380;
  font-size: 20px;
  font-weight: 800;
}
.protection-state.connected { color: #00d09c; }

.ip-line {
  margin-top: 18px;
  color: #ffffff;
  font-size: 15px;
  font-weight: 700;
}

.quick-connect,
.connect-pill {
  width: 100%;
  height: 48px;
  margin-top: 26px;
  border: 0;
  border-radius: 10px;
  color: white;
  background: linear-gradient(135deg, #6e49ff, #8b5cf6);
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.16), 0 14px 30px rgba(103, 70, 255, 0.24);
  font: inherit;
  font-weight: 800;
  cursor: pointer;
}
.quick-connect:disabled,
.connect-pill:disabled {
  cursor: progress;
  opacity: 0.65;
}

.sidebar-tabs {
  display: grid;
  grid-template-columns: 1fr 1fr;
  margin: 10px -22px 20px;
  overflow: hidden;
  border-radius: 0 34px 0 0;
  background: #282431;
}
.sidebar-tabs button,
.mode-strip button,
.section-title button,
.group-title button,
.right-rail button,
.subscribe-dock button {
  border: 0;
  color: inherit;
  background: transparent;
  font: inherit;
  cursor: pointer;
}
.sidebar-tabs button {
  height: 54px;
  color: #a9a3b8;
  font-weight: 800;
}
.sidebar-tabs .is-active {
  color: #fff;
  background: rgba(255, 255, 255, 0.04);
}

.search-box {
  height: 46px;
  padding: 0 16px;
  color: #8f879f;
  border: 1px solid rgba(255, 255, 255, 0.14);
  border-radius: 9px;
  line-height: 46px;
}

.mode-strip {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
  margin: 18px 0;
}
.mode-strip button {
  height: 40px;
  border-radius: 9px;
  color: #bdb7cb;
  background: rgba(255, 255, 255, 0.05);
}
.mode-strip .active {
  color: #fff;
  background: rgba(139, 92, 246, 0.32);
}

.section-title,
.group-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin: 14px 0 8px;
  color: #a9a3b8;
  font-size: 12px;
  font-weight: 800;
}
.group-actions {
  display: inline-flex;
  gap: 8px;
  align-items: center;
}
.group-actions button:disabled {
  cursor: not-allowed;
  opacity: 0.45;
}

.node-row {
  display: grid;
  grid-template-columns: 26px minmax(0, 1fr) auto;
  align-items: center;
  width: 100%;
  min-height: 42px;
  margin-bottom: 6px;
  padding: 0 12px;
  border: 0;
  border-radius: 9px;
  color: #e9e4f5;
  background: transparent;
  font: inherit;
  text-align: left;
  cursor: pointer;
}
.node-row:hover,
.node-row.selected {
  background: rgba(255, 255, 255, 0.07);
}
.node-row.fastest {
  background: rgba(255, 255, 255, 0.06);
}
.flag {
  color: #8b5cf6;
}
.node-label {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.latency {
  color: #35d29c;
  font-size: 12px;
  font-variant-numeric: tabular-nums;
}
.node-meta {
  display: grid;
  justify-items: end;
  gap: 2px;
  min-width: 56px;
  color: #35d29c;
  font-size: 11px;
  font-variant-numeric: tabular-nums;
}
.node-meta small {
  max-width: 92px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.geo-action {
  color: #8b5cf6;
}
.geo-action.loading {
  color: #f5b84c;
}
.empty-nodes {
  padding: 18px 0;
  color: #8f879f;
}

.map-stage {
  position: relative;
  min-height: 680px;
  overflow: hidden;
  background: #0b0820;
}
.map-canvas {
  position: absolute;
  inset: 0;
}
.map-shade {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background:
    radial-gradient(circle at 50% 32%, rgba(124, 92, 255, 0.18), transparent 42%),
    linear-gradient(180deg, rgba(11, 8, 32, 0.55) 0%, rgba(11, 8, 32, 0) 18%, rgba(11, 8, 32, 0) 78%, rgba(11, 8, 32, 0.55) 100%);
}

.status-crown {
  position: absolute;
  top: 0;
  left: 50%;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  width: 300px;
  padding: 18px 0 16px;
  border-radius: 0 0 44px 44px;
  background: rgba(25, 21, 34, 0.9);
  transform: translateX(-50%);
}
.status-crown strong {
  color: #ff6380;
  font-size: 22px;
}
.status-crown strong.connected { color: #00d09c; }
.status-crown span {
  display: grid;
  place-items: center;
  width: 42px;
  height: 42px;
  border-radius: 999px;
  background: #4b465a;
}

.server-hero {
  position: relative;
  z-index: 1;
  width: min(420px, 80%);
  margin: 126px auto 0;
  text-align: center;
}
.server-badge {
  display: grid;
  place-items: center;
  width: 52px;
  height: 52px;
  margin: 0 auto 12px;
  border-radius: 999px;
  background: linear-gradient(135deg, #7c5cff, #00c48c);
  font-size: 26px;
  font-weight: 900;
}
.server-hero h1 {
  margin: 0;
  font-size: 24px;
}
.server-hero p {
  margin: 8px 0 0;
  color: #bcb5cc;
}
.connect-pill {
  width: min(300px, 100%);
  margin-top: 28px;
}

.bottom-status {
  position: absolute;
  left: 28px;
  right: 28px;
  bottom: 28px;
  display: flex;
  gap: 18px;
  align-items: center;
  min-height: 58px;
  padding: 0 22px;
  color: #c6bfce;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 16px;
  background: rgba(18, 15, 25, 0.78);
}
.bottom-status strong {
  color: #00d09c;
}
.bottom-status .danger {
  color: #ff4057;
}
.map-attribution {
  margin-left: auto;
  color: rgba(198, 191, 206, 0.55);
  font-size: 10px;
}

.right-rail {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
  padding: 34px 0;
  background: rgba(18, 15, 25, 0.72);
  border-left: 1px solid rgba(255, 255, 255, 0.08);
}
.right-rail button {
  display: grid;
  gap: 8px;
  justify-items: center;
  width: 72px;
  color: #c9c2d7;
  font-size: 22px;
}
.right-rail span {
  font-size: 11px;
}
.rail-bottom {
  margin-top: auto;
}

.account-dock {
  grid-column: 2 / 4;
  display: grid;
  grid-template-columns: repeat(3, minmax(140px, 1fr)) minmax(240px, 1.3fr);
  gap: 12px;
  padding: 14px 18px;
  background: rgba(18, 15, 25, 0.58);
  border-top: 1px solid rgba(255, 255, 255, 0.08);
}
.dock-card {
  min-width: 0;
  padding: 14px;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.045);
}
.dock-card span {
  display: block;
  color: #9f97ae;
  font-size: 12px;
  font-weight: 800;
}
.dock-card strong {
  display: block;
  margin-top: 8px;
  overflow: hidden;
  color: #fff;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 18px;
}
.traffic-bar {
  height: 7px;
  margin: 12px 0 8px;
  overflow: hidden;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.11);
}
.traffic-bar i {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: linear-gradient(90deg, #7c5cff, #00c48c);
}
.dock-card small {
  color: #bcb5cc;
}
.subscribe-dock {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 110px minmax(0, 1fr) auto;
  align-items: center;
  gap: 10px;
}
.subscribe-dock code {
  min-width: 0;
  overflow: hidden;
  color: #e6e0ef;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.subscribe-dock button {
  height: 32px;
  padding: 0 12px;
  border-radius: 8px;
  color: white;
  background: rgba(139, 92, 246, 0.32);
}

.cue-alert {
  grid-column: 1 / -1;
}
.cue-body {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  flex-wrap: wrap;
}

.tone-success { color: #35d29c; }
.tone-warning { color: #f5b84c; }
.tone-error { color: #ff6380; }
.tone-default { color: #8f879f; }

.log-pre {
  margin: 0;
  padding: 12px;
  background: rgba(127, 127, 127, 0.08);
  border-radius: 6px;
  font-family: ui-monospace, "SFMono-Regular", Menlo, monospace;
  font-size: 11px;
  line-height: 1.5;
  white-space: pre-wrap;
  word-break: break-all;
}

.log-path {
  display: block;
  margin-top: 10px;
  font-size: 11px;
  font-family: ui-monospace, "SFMono-Regular", Menlo, monospace;
  opacity: 0.65;
}

.helper-panel,
.kernel-info,
.helper-paths {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.helper-tip {
  font-size: 12px;
  line-height: 1.55;
}

.helper-paths {
  gap: 4px;
  font-family: ui-monospace, "SFMono-Regular", Menlo, monospace;
  font-size: 11px;
}

@media (max-width: 980px) {
  .proton-workspace {
    grid-template-columns: 1fr;
  }
  .proton-sidebar,
  .right-rail {
    display: none;
  }
  .account-dock {
    grid-column: 1;
    grid-template-columns: 1fr 1fr;
  }
  .map-stage {
    min-height: 620px;
  }
}
</style>
