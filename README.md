const nodes = {
  A: { x: 150, y: 110 },
  B: { x: 255, y: 185 },
  C: { x: 160, y: 300 },
  D: { x: 360, y: 150 },
  E: { x: 390, y: 270 },
  F: { x: 290, y: 400 },
  G: { x: 500, y: 100 },
  H: { x: 585, y: 225 },
  I: { x: 510, y: 390 },
  J: { x: 675, y: 120 },
  K: { x: 710, y: 260 },
  L: { x: 640, y: 425 },
  M: { x: 795, y: 150 },
  N: { x: 820, y: 295 },
  O: { x: 765, y: 430 }
};

const graph = {
  A: { B: 7, C: 6, D: 10 },
  B: { A: 7, D: 8, E: 6, F: 9 },
  C: { A: 6, E: 8, F: 5 },
  D: { A: 10, B: 8, G: 6, E: 7, H: 9 },
  E: { B: 6, C: 8, D: 7, F: 5, H: 8, I: 7 },
  F: { B: 9, C: 5, E: 5, I: 7, L: 9 },
  G: { D: 6, H: 7, J: 6 },
  H: { D: 9, E: 8, G: 7, I: 6, K: 8, J: 7 },
  I: { E: 7, F: 7, H: 6, L: 6, K: 8 },
  J: { G: 6, H: 7, M: 5, K: 6 },
  K: { H: 8, I: 8, J: 6, L: 7, N: 6 },
  L: { F: 9, I: 6, K: 7, O: 8 },
  M: { J: 5, N: 6 },
  N: { K: 6, M: 6, O: 5 },
  O: { L: 8, N: 5 }
};

const baseTraffic = {
  "A-B": 22, "A-C": 40, "A-D": 26,
  "B-D": 68, "B-E": 64, "B-F": 78,
  "C-E": 46, "C-F": 66,
  "D-E": 71, "D-G": 42, "D-H": 57,
  "E-F": 83, "E-H": 66, "E-I": 61,
  "F-I": 70, "F-L": 63,
  "G-H": 38, "G-J": 48,
  "H-I": 78, "H-J": 52, "H-K": 69,
  "I-K": 74, "I-L": 58,
  "J-K": 55, "J-M": 41,
  "K-L": 72, "K-N": 61,
  "L-O": 50,
  "M-N": 38,
  "N-O": 45
};

function edgeKey(a, b) {
  return [a, b].sort().join("-");
}

function trafficForEdge(a, b) {
  const key = edgeKey(a, b);
  const base = baseTraffic[key] ?? 20;
  const drift = Math.sin((a.charCodeAt(0) + b.charCodeAt(0) + Date.now() / 1000) * 4.2) * 11;
  return Math.max(10, Math.min(95, Math.round(base + drift)));
}

function getTrafficState(value) {
  if (value >= 70) return { label: "Heavy", className: "heavy", color: "#ff5f7b" };
  if (value >= 45) return { label: "Moderate", className: "medium", color: "#ffb54a" };
  return { label: "Low", className: "low", color: "#34d79a" };
}

function allRoutes(start, end, maxDepth = 8) {
  const results = [];
  const seen = new Set();

  function dfs(current, target, path, visited) {
    if (path.length > maxDepth) return;
    if (current === target) {
      const key = path.join("->");
      if (!seen.has(key)) {
        seen.add(key);
        results.push([...path]);
      }
      return;
    }

    for (const next of Object.keys(graph[current] || {})) {
      if (visited.has(next)) continue;
      visited.add(next);
      path.push(next);
      dfs(next, target, path, visited);
      path.pop();
      visited.delete(next);
    }
  }

  dfs(start, end, [start], new Set([start]));
  return results;
}

function routeScore(path) {
  let totalScore = 0;
  let congestionSum = 0;

  for (let i = 0; i < path.length - 1; i += 1) {
    const a = path[i];
    const b = path[i + 1];
    const distance = graph[a][b];
    const congestion = trafficForEdge(a, b);
    totalScore += distance * (1 + congestion / 100);
    congestionSum += congestion;
  }

  return {
    score: Number(totalScore.toFixed(1)),
    avgTraffic: Math.round(congestionSum / (path.length - 1))
  };
}

function getRankedRoutes(start, end) {
  return allRoutes(start, end, 8)
    .map((path) => ({ path, ...routeScore(path) }))
    .sort((a, b) => a.score - b.score)
    .slice(0, 4);
}

function drawRoute(path, className) {
  const svg = document.getElementById("mapSvg");
  let d = "";

  path.forEach((node, index) => {
    const { x, y } = nodes[node];
    d += `${index === 0 ? "M" : "L"} ${x} ${y} `;
  });

  const routePath = document.createElementNS("http://www.w3.org/2000/svg", "path");
  routePath.setAttribute("d", d.trim());
  routePath.setAttribute("class", className);
  svg.appendChild(routePath);
}

function renderMap(start, end, rankedRoutes) {
  const svg = document.getElementById("mapSvg");
  svg.innerHTML = "";

  const selectedRoads = new Set();
  rankedRoutes.forEach((route) => {
    for (let i = 0; i < route.path.length - 1; i += 1) {
      selectedRoads.add(edgeKey(route.path[i], route.path[i + 1]));
    }
  });

  Object.keys(graph).forEach((a) => {
    Object.keys(graph[a]).forEach((b) => {
      const x1 = nodes[a].x;
      const y1 = nodes[a].y;
      const x2 = nodes[b].x;
      const y2 = nodes[b].y;
      const traffic = trafficForEdge(a, b);
      const state = getTrafficState(traffic);

      const line = document.createElementNS("http://www.w3.org/2000/svg", "line");
      line.setAttribute("x1", x1);
      line.setAttribute("y1", y1);
      line.setAttribute("x2", x2);
      line.setAttribute("y2", y2);
      line.setAttribute("stroke", state.color);
      line.setAttribute("stroke-width", traffic >= 70 ? 10 : traffic >= 45 ? 8 : 6);
      line.setAttribute("opacity", traffic >= 70 ? 0.9 : traffic >= 45 ? 0.7 : 0.55);
      line.setAttribute("class", "link-base");
      svg.appendChild(line);

      if (selectedRoads.has(edgeKey(a, b))) {
        const glow = document.createElementNS("http://www.w3.org/2000/svg", "line");
        glow.setAttribute("x1", x1);
        glow.setAttribute("y1", y1);
        glow.setAttribute("x2", x2);
        glow.setAttribute("y2", y2);
        glow.setAttribute("stroke", "rgba(255,255,255,0.22)");
        glow.setAttribute("stroke-width", 16);
        glow.setAttribute("opacity", 0.17);
        glow.setAttribute("class", "link-base");
        svg.insertBefore(glow, line);
      }
    });
  });

  if (rankedRoutes[0]) drawRoute(rankedRoutes[0].path, "route-main");
  if (rankedRoutes[1]) drawRoute(rankedRoutes[1].path, "route-alt");

  Object.keys(nodes).forEach((nodeName) => {
    const circle = document.createElementNS("http://www.w3.org/2000/svg", "circle");
    circle.setAttribute("cx", nodes[nodeName].x);
    circle.setAttribute("cy", nodes[nodeName].y);
    circle.setAttribute("r", nodeName === start || nodeName === end ? 15 : 11);
    circle.setAttribute("class", nodeName === start || nodeName === end ? "node is-endpoint" : "node");
    svg.appendChild(circle);

    const label = document.createElementNS("http://www.w3.org/2000/svg", "text");
    label.setAttribute("x", nodes[nodeName].x);
    label.setAttribute("y", nodes[nodeName].y);
    label.setAttribute("class", "label");
    label.textContent = nodeName;
    svg.appendChild(label);
  });
}

function renderRoutes(rankedRoutes) {
  const routeList = document.getElementById("routeList");
  routeList.innerHTML = "";

  rankedRoutes.forEach((route, index) => {
    const card = document.createElement("div");
    card.className = `route-card${index === 0 ? " active" : ""}`;
    card.innerHTML = `
      <div class="route-head">
        <div class="route-title">${index === 0 ? "Best route" : index === 1 ? "Alternate route" : "Backup route"}</div>
        <div class="route-score">${route.score.toFixed(1)} pts</div>
      </div>
      <div class="route-path">${route.path.join(" → ")}</div>
    `;
    routeList.appendChild(card);
  });
}

function renderInsights(rankedRoutes) {
  const insights = document.getElementById("insights");
  insights.innerHTML = "";

  const main = rankedRoutes[0];
  const alt = rankedRoutes[1];
  const segments = [];

  main.path.forEach((node, index) => {
    if (index >= main.path.length - 1) return;
    const a = node;
    const b = main.path[index + 1];
    segments.push({ pair: `${a}-${b}`, value: trafficForEdge(a, b) });
  });

  segments.sort((a, b) => b.value - a.value);
  const topHotspot = segments[0] || { pair: "main corridor", value: 0 };

  const insightText = alt
    ? `The main route is recommended, but the alternate pathway via ${alt.path[1]} is the safer option if congestion rises near ${topHotspot.pair}.`
    : `Conditions are manageable, but watch for traffic build-up near ${topHotspot.pair}.`;

  const card = document.createElement("div");
  card.className = "route-card active";
  card.innerHTML = `
    <div class="route-head">
      <div class="route-title">Traffic insight</div>
      <div class="route-score">${main.avgTraffic}%</div>
    </div>
    <div class="route-path">${insightText}</div>
  `;
  insights.appendChild(card);
}

function renderHotspots(rankedRoutes) {
  const hotspotList = document.getElementById("hotspotList");
  hotspotList.innerHTML = "";

  const main = rankedRoutes[0];
  const entries = [];

  main.path.forEach((node, index) => {
    if (index >= main.path.length - 1) return;
    const a = node;
    const b = main.path[index + 1];
    entries.push({ pair: `${a}-${b}`, value: trafficForEdge(a, b) });
  });

  entries.sort((a, b) => b.value - a.value);
  entries.slice(0, 4).forEach((entry) => {
    const state = getTrafficState(entry.value);
    const row = document.createElement("div");
    row.className = "hotspot";
    row.innerHTML = `
      <div class="hotspot-main">
        <strong>${entry.pair}</strong>
        <span>${state.label} traffic</span>
      </div>
      <span class="pill ${state.className}">${entry.value}%</span>
    `;
    hotspotList.appendChild(row);
  });
}

function updateStatus(avgTraffic) {
  const badge = document.getElementById("statusBadge");

  if (avgTraffic >= 70) {
    badge.textContent = "Heavy traffic";
    badge.className = "status bad";
  } else if (avgTraffic >= 45) {
    badge.textContent = "Moderate flow";
    badge.className = "status warn";
  } else {
    badge.textContent = "Route stable";
    badge.className = "status good";
  }
}

function populateSelectors() {
  const fromSelect = document.getElementById("fromNode");
  const toSelect = document.getElementById("toNode");
  const ids = Object.keys(nodes);

  ids.forEach((id) => {
    const fromOption = document.createElement("option");
    fromOption.value = id;
    fromOption.textContent = id;
    fromSelect.appendChild(fromOption);

    const toOption = document.createElement("option");
    toOption.value = id;
    toOption.textContent = id;
    toSelect.appendChild(toOption);
  });

  fromSelect.value = "A";
  toSelect.value = "O";
}

function analyzeRoute() {
  const from = document.getElementById("fromNode").value;
  const to = document.getElementById("toNode").value;

  if (from === to) {
    window.alert("Please choose two different places before analyzing the route.");
    return;
  }

  const rankedRoutes = getRankedRoutes(from, to);
  if (!rankedRoutes.length) {
    window.alert("No valid route was found for this trip.");
    return;
  }

  renderMap(from, to, rankedRoutes);
  renderRoutes(rankedRoutes);
  renderInsights(rankedRoutes);
  renderHotspots(rankedRoutes);

  document.getElementById("avgTraffic").textContent = `${rankedRoutes[0].avgTraffic}%`;
  document.getElementById("bestTime").textContent = `${rankedRoutes[0].score.toFixed(0)} min`;
  updateStatus(rankedRoutes[0].avgTraffic);
}

populateSelectors();
document.getElementById("analyzeBtn").addEventListener("click", analyzeRoute);
document.getElementById("fromNode").addEventListener("change", analyzeRoute);
document.getElementById("toNode").addEventListener("change", analyzeRoute);
analyzeRoute();
