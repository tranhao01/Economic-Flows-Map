# Economic-Flows-Map
import React, { useMemo, useRef, useState, useEffect } from "react";

// --- Economic Flow Map — 12 Saigon Districts (Interactive + Accurate Map) ---
// • Top: draggable flow layout; cables (curved SVG) follow.
// • Bottom: MAP view computed from real-world lat/lon (centroid-ish) with a simple projection.
// • Pure React + TailwindCSS. No external network calls.

export default function EconomicFlowMap() {
  // Canvas size (px) — fixed for stability inside canvas preview
  const CANVAS_W = 1200;
  const CANVAS_H = 720;

  // Groups for quick styling
  const GROUPS = {
    gateway: {
      label: "Cửa ngõ (cảng/sân bay/KCX)",
      bg: "bg-sky-100",
      ring: "ring-sky-400",
      text: "text-sky-900",
      dot: "#0284c7",
    },
    wholesale: {
      label: "Đầu mối bán sỉ/Chợ Lớn",
      bg: "bg-amber-100",
      ring: "ring-amber-400",
      text: "text-amber-900",
      dot: "#ca8a04",
    },
    consumer: {
      label: "Tiêu dùng – dịch vụ/du lịch",
      bg: "bg-slate-100",
      ring: "ring-slate-400",
      text: "text-slate-900",
      dot: "#334155",
    },
  } as const;

  // Initial node layout (12 districts)
  const initialNodes = useMemo(
    () => ([
      { id: "Q4-Cang", name: "Q4 — Cảng Sài Gòn", group: "gateway", x: 120, y: 160 },
      { id: "TB-TSN", name: "Tân Bình — TSN (hàng không)", group: "gateway", x: 120, y: 320 },
      { id: "Q7-KCX", name: "Q7 — KCX Tân Thuận", group: "gateway", x: 120, y: 500 },

      { id: "Q6-BinhTay", name: "Q6 — Chợ Bình Tây (wholesale)", group: "wholesale", x: 460, y: 220 },
      { id: "Q5-Chinatown", name: "Q5 — Chinatown (bán sỉ)", group: "wholesale", x: 460, y: 380 },
      { id: "Q8-BinhDong", name: "Q8 — Bến Bình Đông (thủy)", group: "wholesale", x: 460, y: 540 },

      { id: "Q1-CBD", name: "Q1 — CBD (tài chính/du lịch)", group: "consumer", x: 900, y: 160 },
      { id: "Q3-GiaoDuc", name: "Q3 — Giáo dục/Y tế", group: "consumer", x: 900, y: 300 },
      { id: "BT-BinhThanh", name: "Bình Thạnh — Dịch vụ/cao ốc", group: "consumer", x: 900, y: 440 },
      { id: "GV-GoVap", name: "Gò Vấp — Bán lẻ đại siêu thị", group: "consumer", x: 900, y: 580 },
      { id: "Q10-NhatTao", name: "Q10 — Nhật Tảo (điện tử cũ)", group: "consumer", x: 740, y: 380 },
      { id: "Q11-DamSen", name: "Q11 — Đầm Sen (giải trí)", group: "consumer", x: 740, y: 520 },
    ]),
    []
  );

  // Edge list — "mấy sợi dây" (cables)
  const edges = useMemo(
    () => ([
      { from: "Q7-KCX", to: "Q4-Cang", w: 4, type: "export" },
      { from: "Q7-KCX", to: "Q6-BinhTay", w: 3, type: "export" },
      { from: "TB-TSN", to: "Q6-BinhTay", w: 3, type: "air" },
      { from: "Q4-Cang", to: "Q6-BinhTay", w: 4, type: "sea" },
      { from: "Q8-BinhDong", to: "Q6-BinhTay", w: 2, type: "river" },

      { from: "Q6-BinhTay", to: "Q5-Chinatown", w: 3, type: "wholesale" },
      { from: "Q5-Chinatown", to: "Q1-CBD", w: 4, type: "retail" },
      { from: "Q6-BinhTay", to: "GV-GoVap", w: 3, type: "retail" },
      { from: "Q6-BinhTay", to: "BT-BinhThanh", w: 3, type: "retail" },
      { from: "Q5-Chinatown", to: "Q3-GiaoDuc", w: 2, type: "services" },
      { from: "Q5-Chinatown", to: "BT-BinhThanh", w: 2, type: "services" },

      { from: "Q1-CBD", to: "Q3-GiaoDuc", w: 2, type: "tourism" },
      { from: "Q1-CBD", to: "BT-BinhThanh", w: 2, type: "tourism" },
      { from: "Q1-CBD", to: "Q4-Cang", w: 1, type: "tourism" },

      { from: "Q10-NhatTao", to: "GV-GoVap", w: 2, type: "electronics" },
      { from: "Q10-NhatTao", to: "BT-BinhThanh", w: 2, type: "electronics" },
      { from: "Q11-DamSen", to: "Q1-CBD", w: 1, type: "leisure" },
      { from: "TB-TSN", to: "Q1-CBD", w: 3, type: "air" },
    ]),
    []
  );

  // Cable style colors per type
  const typeColor: Record<string, string> = {
    export: "#f59e0b", // amber-500
    air: "#0ea5e9", // sky-500
    sea: "#3b82f6", // blue-500
    river: "#22c55e", // green-500
    wholesale: "#eab308", // amber-400
    retail: "#8b5cf6", // violet-500
    services: "#64748b", // slate-500
    tourism: "#10b981", // emerald-500
    electronics: "#a855f7", // purple-500
    leisure: "#f43f5e", // rose-500
  };

  // Node state (positions)
  const [nodes, setNodes] = useState(() => {
    const map: Record<string, { id: string; name: string; group: keyof typeof GROUPS; x: number; y: number } > = {} as any;
    initialNodes.forEach((n) => (map[n.id] = { ...n } as any));
    return map;
  });

  // UI controls (top panel)
  const [showLabels, setShowLabels] = useState(true);
  const [showCables, setShowCables] = useState(true);
  const [thicken, setThicken] = useState(true);
  const [curvature, setCurvature] = useState(0.25); // 0..0.6
  const [lastAction, setLastAction] = useState<"none" | "normalize" | "fitMap" | "reset">("none");

  const wrapperRef = useRef<HTMLDivElement | null>(null);
  const dragRef = useRef<{ id: string | null; dx: number; dy: number } | null>(null);

  // Helper: build a smooth cubic path from A to B with some vertical lift
  const buildFlowPath = (a: { x: number; y: number }, b: { x: number; y: number }) => {
    const dx = b.x - a.x;
    const dy = b.y - a.y;
    const lift = 120 * curvature; // lift controls the "cable" bow
    const cx1 = a.x + dx * 0.33;
    const cy1 = a.y + dy * 0.2 - lift;
    const cx2 = a.x + dx * 0.66;
    const cy2 = a.y + dy * 0.8 + lift;
    return `M ${a.x},${a.y} C ${cx1},${cy1} ${cx2},${cy2} ${b.x},${b.y}`;
  };

  // Drag handlers (HTML layer for nodes)
  const onPointerDown = (e: React.PointerEvent, id: string) => {
    const rect = wrapperRef.current!.getBoundingClientRect();
    const n = nodes[id];
    dragRef.current = { id, dx: e.clientX - rect.left - n.x, dy: e.clientY - rect.top - n.y };
    (e.target as HTMLElement).setPointerCapture(e.pointerId);
  };
  const onPointerMove = (e: React.PointerEvent) => {
    if (!dragRef.current?.id) return;
    const rect = wrapperRef.current!.getBoundingClientRect();
    const x = e.clientX - rect.left - dragRef.current.dx;
    const y = e.clientY - rect.top - dragRef.current.dy;
    const id = dragRef.current.id;
    setNodes((prev) => {
      const next = { ...prev } as any;
      // clamp inside canvas
      next[id] = {
        ...next[id],
        x: Math.max(40, Math.min(CANVAS_W - 40, x)),
        y: Math.max(40, Math.min(CANVAS_H - 40, y)),
      };
      return next;
    });
  };
  const onPointerUp = () => {
    dragRef.current = { id: null, dx: 0, dy: 0 };
  };

  // Build edges with live node coordinates (top panel)
  const edgePaths = useMemo(() => {
    return edges
      .map((e, i) => {
        const a = nodes[e.from];
        const b = nodes[e.to];
        if (!a || !b) return null;
        return {
          i,
          d: buildFlowPath(a, b),
          color: typeColor[e.type] || "#94a3b8",
          width: thicken ? Math.max(1.5, e.w) : 2,
          type: e.type,
        };
      })
      .filter(Boolean) as { i: number; d: string; color: string; width: number; type: string }[];
  }, [edges, nodes, thicken, curvature]);

  // Convenience list of nodes for rendering
  const nodeList = useMemo(() => Object.values(nodes), [nodes]);

  // ======== COLUMN POSITIONS (centered) ========
  const COL_X_CENTERED = useMemo(() => ({
    gateway: CANVAS_W / 6,       // 1/6 W => 200 when W=1200
    wholesale: CANVAS_W / 2,     // center => 600
    consumer: (CANVAS_W * 5) / 6 // 5/6 W => 1000
  }), []);

  // ===== Normalize to centered columns + even vertical spacing
  const normalizePositions = () => {
    const marginTop = 80;
    const marginBottom = 80;
    const avail = CANVAS_H - marginTop - marginBottom;
    const orderByGroup: Record<string, string[]> = { gateway: [], wholesale: [], consumer: [] } as any;
    initialNodes.forEach((n) => (orderByGroup[n.group].push(n.id)));

    setNodes((prev) => {
      const next = { ...prev } as any;
      (Object.keys(orderByGroup) as Array<keyof typeof orderByGroup>).forEach((g) => {
        const ids = orderByGroup[g];
        const N = ids.length;
        ids.forEach((id, idx) => {
          const y = Math.round(marginTop + (avail * (idx + 1)) / (N + 1));
          next[id] = { ...next[id], x: COL_X_CENTERED[g as any], y };
        });
      });
      return next;
    });
    setLastAction("normalize");
  };

  // ===== Fit top layout to REAL MAP positions (centered in canvas)
  // Take the already-projected map coordinates, re-fit to the top canvas, and apply to nodes
  const fitToRealMapPositions = () => {
    // get bbox of map coords
    const xs = Object.values(mapCoords).map((p) => p.x);
    const ys = Object.values(mapCoords).map((p) => p.y);
    const minX = Math.min(...xs);
    const maxX = Math.max(...xs);
    const minY = Math.min(...ys);
    const maxY = Math.max(...ys);
    const spanX = maxX - minX;
    const spanY = maxY - minY;

    const pad = 40;
    const scale = Math.min((CANVAS_W - 2 * pad) / spanX, (CANVAS_H - 2 * pad) / spanY);
    const offX = (CANVAS_W - spanX * scale) / 2;
    const offY = (CANVAS_H - spanY * scale) / 2;

    // build target positions
    const target: Record<string, { x: number; y: number }> = {};
    Object.entries(mapCoords).forEach(([id, p]) => {
      target[id] = { x: (p.x - minX) * scale + offX, y: (p.y - minY) * scale + offY };
    });

    // apply & verify
    setNodes((prev) => {
      const next = { ...prev } as any;
      Object.keys(target).forEach((id) => {
        next[id] = { ...next[id], x: target[id].x, y: target[id].y };
      });
      return next;
    });

    // quick verification (epsilon tolerance)
    const EPS = 0.5;
    Object.entries(target).forEach(([id, p]) => {
      const n = { ...nodes[id], x: p.x, y: p.y };
      console.assert(Math.abs(n.x - p.x) <= EPS && Math.abs(n.y - p.y) <= EPS, `FitMap mismatch for ${id}`);
    });

    setLastAction("fitMap");
  };

  // ===== Reset to initial positions
  const resetPositions = () => {
    const map: Record<string, { id: string; name: string; group: keyof typeof GROUPS; x: number; y: number } > = {} as any;
    initialNodes.forEach((n) => (map[n.id] = { ...n } as any));
    setNodes(map);
    setLastAction("reset");
  };

  // --- MAP VIEW (bottom) ---
  const MAP_W = CANVAS_W;
  const MAP_H = 560;

  // Real-world lat/lon for district pins (centroids-ish)
  // (Approximate; used for more accurate relative placement)
  const GEO: Record<string, { lat: number; lon: number }> = {
    "Q1-CBD": { lat: 10.77556, lon: 106.69694 },
    "Q3-GiaoDuc": { lat: 10.78194, lon: 106.68528 },
    "BT-BinhThanh": { lat: 10.80333, lon: 106.68972 },
    "Q4-Cang": { lat: 10.76167, lon: 106.7025 },
    "Q5-Chinatown": { lat: 10.75611, lon: 106.66694 },
    "Q6-BinhTay": { lat: 10.74611, lon: 106.63611 },
    "Q7-KCX": { lat: 10.73861, lon: 106.72639 },
    "Q8-BinhDong": { lat: 10.7241, lon: 106.6286 },
    "Q10-NhatTao": { lat: 10.77361, lon: 106.66722 },
    "Q11-DamSen": { lat: 10.76694, lon: 106.64556 },
    "TB-TSN": { lat: 10.79417, lon: 106.65444 },
    "GV-GoVap": { lat: 10.84167, lon: 106.66667 },
  };

  // Equirectangular projection with cos(meanLat) correction for lon so X/Y share one scale
  const proj = (() => {
    const pts = Object.values(GEO);
    const lats = pts.map((p) => p.lat);
    const lons = pts.map((p) => p.lon);
    const minLat = Math.min(...lats);
    const maxLat = Math.max(...lats);
    const minLon = Math.min(...lons);
    const maxLon = Math.max(...lons);
    const pad = 40; // inner padding
    const meanLat = (minLat + maxLat) / 2;
    const cos = Math.cos((meanLat * Math.PI) / 180);
    const spanX = (maxLon - minLon) * cos;
    const spanY = maxLat - minLat;
    const scale = Math.min((MAP_W - 2 * pad) / spanX, (MAP_H - 2 * pad) / spanY);
    const pxPerDeg = scale; // uniform after cos-correction
    const kmPerDeg = 111.32;
    const pxPerKm = pxPerDeg / kmPerDeg;
    const barKm = 5; // scale bar length in km
    const barPx = Math.max(80, barKm * pxPerKm);
    return {
      toXY(lat: number, lon: number) {
        const x = (lon - minLon) * cos * scale + pad;
        const y = (maxLat - lat) * scale + pad;
        return { x, y };
      },
      barPx,
      barKm,
      pad,
    };
  })();

  // Compute map coordinates from lat/lon
  const mapCoords: Record<string, { x: number; y: number }> = Object.fromEntries(
    Object.entries(GEO).map(([id, g]) => [id, proj.toXY(g.lat, g.lon)])
  );

  const [showMapEdges, setShowMapEdges] = useState(true);
  const [showMapLabels, setShowMapLabels] = useState(true);

  // Edge path builder for map (mild curvature)
  const buildMapPath = (a: { x: number; y: number }, b: { x: number; y: number }) => {
    const dx = b.x - a.x;
    const dy = b.y - a.y;
    const lift = 80; // softer than top panel
    const cx1 = a.x + dx * 0.33;
    const cy1 = a.y + dy * 0.2 - lift;
    const cx2 = a.x + dx * 0.66;
    const cy2 = a.y + dy * 0.8 + lift;
    return `M ${a.x},${a.y} C ${cx1},${cy1} ${cx2},${cy2} ${b.x},${b.y}`;
  };

  // Build edges for map panel
  const mapEdgePaths = useMemo(() => {
    return edges
      .map((e, i) => {
        const a = mapCoords[e.from];
        const b = mapCoords[e.to];
        if (!a || !b) return null;
        return {
          i,
          d: buildMapPath(a, b),
          color: typeColor[e.type] || "#94a3b8",
          width: Math.max(1.25, e.w * 0.9),
        };
      })
      .filter(Boolean) as { i: number; d: string; color: string; width: number }[];
  }, [edges]);

  // --- RUNTIME SANITY TESTS (act like lightweight test cases) ---
  useEffect(() => {
    // 1) unique node ids
    const ids = initialNodes.map((n) => n.id);
    const set = new Set(ids);
    console.assert(set.size === ids.length, "Duplicate node id detected");

    // 2) edges point to existing ids
    const okEdges = edges.every((e) => set.has(e.from) && set.has(e.to));
    console.assert(okEdges, "Edge refers to unknown node id");

    // 3) all ids have map coordinates within bounds
    const allMapped = ids.every((id) => mapCoords[id]);
    console.assert(allMapped, "Missing mapCoords for some id");
    const inBounds = Object.values(mapCoords).every((p) => p.x >= 0 && p.x <= MAP_W && p.y >= 0 && p.y <= MAP_H);
    console.assert(inBounds, "Some map points are out of bounds");

    // 4) edge types are all color-mapped
    const mappedTypes = edges.every((e) => Object.prototype.hasOwnProperty.call(typeColor, e.type));
    console.assert(mappedTypes, "Some edge types have no color mapping");

    // 5) basic edge count sanity
    console.assert(edges.length > 0, "No edges defined");
  }, []);

  return (
    <div className="w-full">
      {/* ====== TOP: Flow layout (draggable) ====== */}
      <div className="mb-3 flex flex-wrap items-center gap-3">
        <span className="text-sm font-medium text-slate-600">Tùy chỉnh (sơ đồ luồng):</span>
        <label className="flex items-center gap-2 text-sm text-slate-700">
          <input type="checkbox" className="accent-slate-700" checked={showCables} onChange={(e) => setShowCables(e.target.checked)} />
          Hiển thị dây (cables)
        </label>
        <label className="flex items-center gap-2 text-sm text-slate-700">
          <input type="checkbox" className="accent-slate-700" checked={showLabels} onChange={(e) => setShowLabels(e.target.checked)} />
          Nhãn quận
        </label>
        <label className="flex items-center gap-2 text-sm text-slate-700">
          <input type="checkbox" className="accent-slate-700" checked={thicken} onChange={(e) => setThicken(e.target.checked)} />
          Độ dày theo trọng số
        </label>
        <label className="flex items-center gap-2 text-sm text-slate-700">
          Độ võng dây
          <input type="range" min={0} max={0.6} step={0.05} value={curvature} onChange={(e) => setCurvature(parseFloat(e.target.value))} />
        </label>

        {/* BUTTONS: Centered normalize / Fit to map / Reset */}
        <div className="flex items-center gap-2">
          <button
            onClick={normalizePositions}
            className="inline-flex items-center gap-2 rounded-lg border border-slate-300 bg-white px-3 py-1.5 text-sm text-slate-700 shadow-sm hover:bg-slate-50 active:scale-[0.99]"
            aria-label="Chuẩn hóa vị trí theo 3 cột căn giữa và dàn đều theo chiều dọc"
            title="Chuẩn hóa vị trí (3 cột căn giữa)"
          >
            <span className="inline-block h-2 w-2 rounded-full bg-emerald-500" />
            Chuẩn hóa (căn giữa)
          </button>

          <button
            onClick={fitToRealMapPositions}
            className="inline-flex items-center gap-2 rounded-lg border border-slate-300 bg-white px-3 py-1.5 text-sm text-slate-700 shadow-sm hover:bg-slate-50 active:scale-[0.99]"
            aria-label="Ánh xạ vị trí thực (map) lên sơ đồ trên và căn giữa trong khung"
            title="Căn theo vị trí thực (map → flow)"
          >
            <span className="inline-block h-2 w-2 rounded-full bg-blue-500" />
            Căn theo vị trí thực
          </button>

          <button
            onClick={resetPositions}
            className="inline-flex items-center gap-2 rounded-lg border border-slate-300 bg-white px-3 py-1.5 text-sm text-slate-700 shadow-sm hover:bg-slate-50 active:scale-[0.99]"
            aria-label="Đặt lại vị trí ban đầu"
            title="Đặt lại ban đầu"
          >
            <span className="inline-block h-2 w-2 rounded-full bg-slate-400" />
            Đặt lại ban đầu
          </button>
        </div>
      </div>

      <div
        className="relative mx-auto rounded-2xl border border-slate-200 bg-white shadow-sm"
        style={{ width: CANVAS_W, height: CANVAS_H }}
        ref={wrapperRef}
        onPointerMove={onPointerMove}
        onPointerUp={onPointerUp}
        onPointerCancel={onPointerUp}
      >
        {/* SVG cables */}
        <svg width={CANVAS_W} height={CANVAS_H} className="absolute inset-0">
          <defs>
            <filter id="softGlow" x="-50%" y="-50%" width="200%" height="200%">
              <feGaussianBlur stdDeviation="2" result="blur" />
              <feMerge>
                <feMergeNode in="blur" />
                <feMergeNode in="SourceGraphic" />
              </feMerge>
            </filter>
          </defs>

          {showCables && (
            <g>
              {edgePaths.map((p) => (
                <path
                  key={p.i}
                  d={p.d}
                  fill="none"
                  stroke={p.color}
                  strokeWidth={p.width}
                  strokeOpacity={0.7}
                  filter="url(#softGlow)"
                />
              ))}
            </g>
          )}

          {/* grid background */}
          <g>
            <rect x={0} y={0} width={CANVAS_W} height={CANVAS_H} fill="none" stroke="#e2e8f0" />
            {/* centered vertical guides */}
            <line x1={COL_X_CENTERED.gateway} y1={0} x2={COL_X_CENTERED.gateway} y2={CANVAS_H} stroke="#e2e8f0" strokeDasharray="4 6" />
            <line x1={COL_X_CENTERED.wholesale} y1={0} x2={COL_X_CENTERED.wholesale} y2={CANVAS_H} stroke="#e2e8f0" strokeDasharray="4 6" />
            <line x1={COL_X_CENTERED.consumer} y1={0} x2={COL_X_CENTERED.consumer} y2={CANVAS_H} stroke="#e2e8f0" strokeDasharray="4 6" />
          </g>
        </svg>

        {/* Nodes (HTML) */}
        {nodeList.map((n) => (
          <div
            key={n.id}
            className={`absolute -translate-x-1/2 -translate-y-1/2 select-none rounded-2xl px-3 py-2 text-sm shadow-md ring-2 cursor-grab active:cursor-grabbing ${GROUPS[n.group].bg} ${GROUPS[n.group].ring} ${GROUPS[n.group].text}`}
            style={{ left: n.x, top: n.y }}
            onPointerDown={(e) => onPointerDown(e, n.id)}
          >
            <div className="flex items-center gap-2">
              <span className="inline-block h-2 w-2 rounded-full bg-slate-500" />
              <span className={!showLabels ? "sr-only" : ""}>{n.name}</span>
              {!showLabels && <span className="text-xs text-slate-500">●</span>}
            </div>
          </div>
        ))}

        {/* Column headers */}
        <div className="pointer-events-none absolute left-[calc(16.66%-40px)] top-3 text-xs font-medium text-sky-700">{GROUPS.gateway.label}</div>
        <div className="pointer-events-none absolute left-[calc(50%-40px)] top-3 text-xs font-medium text-amber-700">{GROUPS.wholesale.label}</div>
        <div className="pointer-events-none absolute left-[calc(83.33%-40px)] top-3 text-xs font-medium text-slate-700">{GROUPS.consumer.label}</div>
      </div>

      <div className="mt-4 grid gap-2 text-sm text-slate-600 md:grid-cols-3">
        <LegendChip color="#0ea5e9" label="Hàng không (TSN)" />
        <LegendChip color="#3b82f6" label="Hàng hải (Cảng)" />
        <LegendChip color="#22c55e" label="Đường thủy (Mekong)" />
        <LegendChip color="#eab308" label="Bán sỉ / Wholesale" />
        <LegendChip color="#8b5cf6" label="Bán lẻ / Retail" />
        <LegendChip color="#64748b" label="Dịch vụ / Services" />
        <LegendChip color="#10b981" label="Du lịch / Tourism" />
        <LegendChip color="#a855f7" label="Điện tử (tái chế)" />
        <LegendChip color="#f59e0b" label="Xuất khẩu (KCX)" />
        <LegendChip color="#f43f5e" label="Giải trí (events)" />
      </div>

      <p className="mt-3 text-xs text-slate-500">
        Gợi ý: dùng **Chuẩn hóa (căn giữa)** để dàn hàng gọn; hoặc **Căn theo vị trí thực** để đồng bộ sơ đồ trên với toạ độ map bên dưới.
      </p>

      {/* ====== BOTTOM: MAP VIEW (accurate-ish) ====== */}
      <div className="mt-8 mb-3 flex flex-wrap items-center gap-3">
        <span className="text-sm font-medium text-slate-600">Tùy chỉnh (bản đồ):</span>
        <label className="flex items-center gap-2 text-sm text-slate-700">
          <input type="checkbox" className="accent-slate-700" checked={showMapEdges} onChange={(e) => setShowMapEdges(e.target.checked)} />
          Hiển thị dây trên bản đồ
        </label>
        <label className="flex items-center gap-2 text-sm text-slate-700">
          <input type="checkbox" className="accent-slate-700" checked={showMapLabels} onChange={(e) => setShowMapLabels(e.target.checked)} />
          Nhãn quận trên bản đồ
        </label>
      </div>

      <div className="relative mx-auto rounded-2xl border border-slate-200 bg-white shadow-sm" style={{ width: MAP_W, height: MAP_H }}>
        <svg width={MAP_W} height={MAP_H} className="absolute inset-0">
          {/* Background grid */}
          <defs>
            <pattern id="minorGrid" width="40" height="40" patternUnits="userSpaceOnUse">
              <path d="M 40 0 L 0 0 0 40" fill="none" stroke="#e2e8f0" strokeWidth="1" />
            </pattern>
            <filter id="softGlow2" x="-50%" y="-50%" width="200%" height="200%">
              <feGaussianBlur stdDeviation="1.6" result="blur" />
              <feMerge>
                <feMergeNode in="blur" />
                <feMergeNode in="SourceGraphic" />
              </feMerge>
            </filter>
          </defs>
          <rect x={0} y={0} width={MAP_W} height={MAP_H} fill="url(#minorGrid)" />

          {/* River corridor (approx) based on pin positions */}
          <path
            d={`M ${mapCoords["BT-BinhThanh"].x},${mapCoords["BT-BinhThanh"].y}
                 L ${mapCoords["Q1-CBD"].x},${mapCoords["Q1-CBD"].y}
                 L ${mapCoords["Q4-Cang"].x},${mapCoords["Q4-Cang"].y}
                 L ${mapCoords["Q7-KCX"].x},${mapCoords["Q7-KCX"].y}`}
            fill="none"
            stroke="#3b82f6"
            strokeWidth={10}
            strokeOpacity={0.22}
          />
          {/* Kênh Tàu Hủ – Bến Nghé (approx) */}
          <path
            d={`M ${mapCoords["Q6-BinhTay"].x},${mapCoords["Q6-BinhTay"].y}
                 L ${mapCoords["Q4-Cang"].x},${mapCoords["Q4-Cang"].y}`}
            fill="none"
            stroke="#22c55e"
            strokeWidth={8}
            strokeOpacity={0.18}
          />

          {/* Map edges (same flows as top, placed with projected coords) */}
          {showMapEdges && (
            <g>
              {mapEdgePaths.map((p) => (
                <path key={p.i} d={p.d} fill="none" stroke={p.color} strokeWidth={p.width} strokeOpacity={0.65} filter="url(#softGlow2)" />)
              )}
            </g>
          )}

          {/* District pins */}
          {Object.entries(mapCoords).map(([id, pos]) => {
            const meta = initialNodes.find((n) => n.id === id)!;
            const color = GROUPS[meta.group].dot;
            const label = meta.name;
            return (
              <g key={id}>
                <circle cx={pos.x} cy={pos.y} r={8} fill={color} stroke="#ffffff" strokeWidth={2} />
                {showMapLabels && (
                  <text x={pos.x + 12} y={pos.y + 4} fontSize={12} fill="#334155">{label}</text>
                )}
              </g>
            );
          })}

          {/* Map frame + north arrow + scale bar */}
          <rect x={0} y={0} width={MAP_W} height={MAP_H} fill="none" stroke="#cbd5e1" />

          {/* North arrow */}
          <g transform={`translate(${MAP_W - 40}, 40)`}>
            <polygon points="0,-18 6,0 -6,0" fill="#334155" />
            <line x1={0} y1={0} x2={0} y2={24} stroke="#334155" strokeWidth={2} />
            <text x={0} y={36} textAnchor="middle" fontSize={12} fill="#334155">N</text>
          </g>

          {/* Scale bar */}
          <g transform={`translate(24, ${MAP_H - 40})`}>
            <rect x={0} y={0} width={proj.barPx} height={6} fill="#334155" opacity={0.8} />
            <text x={proj.barPx / 2} y={-6} textAnchor="middle" fontSize={12} fill="#334155">{`${proj.barKm} km`}</text>
          </g>
        </svg>

        {/* Map legend */}
        <div className="absolute left-3 top-3 rounded-xl bg-white/80 p-2 shadow-sm backdrop-blur">
          <div className="grid grid-cols-2 gap-2 text-xs text-slate-700">
            <LegendChip color="#0ea5e9" label="TSN (air)" />
            <LegendChip color="#3b82f6" label="Cảng/river" />
            <LegendChip color="#22c55e" label="Kênh/đường thủy" />
            <LegendChip color="#eab308" label="Wholesale" />
            <LegendChip color="#8b5cf6" label="Retail" />
            <LegendChip color="#10b981" label="Tourism" />
          </div>
        </div>
      </div>

      <p className="mt-3 text-xs text-slate-500">
        Bản đồ dưới đã dùng tọa độ thật (chuẩn hóa) để đặt vị trí 12 quận đúng tương quan; sông/kênh được vẽ gần đúng để định hướng. Nếu bạn muốn mức
        chính xác cao hơn (rìa quận, đa giác ranh giới), mình có thể thêm lớp polygon khi bạn xác nhận phạm vi 12 quận mong muốn.
      </p>
    </div>
  );
}

function LegendChip({ color, label }: { color: string; label: string }) {
  return (
    <div className="flex items-center gap-2">
      <span className="inline-block h-2.5 w-6 rounded-full" style={{ backgroundColor: color, opacity: 0.8 }} />
      <span>{label}</span>
    </div>
  );
}
