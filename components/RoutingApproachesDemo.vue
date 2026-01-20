<script setup>
import { ref, computed } from "vue";

// Same people data as other demos
const allPeople = [
	"Alberto",
	"Alessandro",
	"Alex",
	"Alexander",
	"Ana",
	"Andrei",
	"Attila",
	"Cristiano",
	"Daniele",
	"Davide",
	"Enrico",
	"Federico",
	"Fernando",
	"Gianluca",
	"Jerome",
	"Nicola",
	"Riccardo",
	"Snigdha",
	"SommoMicc",
	"Stefano C.",
	"Stefano D.",
	"Ugo",
	"Wouter",
	"Zak",
];

// Partition colors (consistent with other demos)
const partitionColors = [
	"#89b4fa",
	"#a6e3a1",
	"#f9e2af",
	"#cba6f7",
	"#f38ba8",
	"#94e2d5",
	"#fab387",
	"#74c7ec",
];

// Fixed partition assignment (4 partitions across 3 nodes)
const partitions = [
	{
		id: 0,
		nodeId: 0,
		people: ["Alberto", "Alessandro", "Alex", "Alexander", "Ana", "Andrei"],
	},
	{
		id: 1,
		nodeId: 1,
		people: ["Attila", "Cristiano", "Daniele", "Davide", "Enrico", "Federico"],
	},
	{
		id: 2,
		nodeId: 2,
		people: ["Fernando", "Gianluca", "Jerome", "Nicola", "Riccardo", "Snigdha"],
	},
	{
		id: 3,
		nodeId: 2,
		people: ["SommoMicc", "Stefano C.", "Stefano D.", "Ugo", "Wouter", "Zak"],
	},
];

const nodes = [
	{ id: 0, name: "Node 1" },
	{ id: 1, name: "Node 2" },
	{ id: 2, name: "Node 3" },
];

// Current approach (1, 2, or 3)
const currentApproach = ref(1);

// Animation state
const isAnimating = ref(false);
const animationPhase = ref("idle"); // idle, sending, forwarding, responding
const currentRequest = ref(null);
const requestPath = ref([]);
const highlightedNode = ref(null);
const highlightedPartition = ref(null);
const hops = ref(0);

// Find which partition and node has a person
const findPersonLocation = (name) => {
	for (const p of partitions) {
		if (p.people.includes(name)) {
			return { partition: p, node: nodes[p.nodeId] };
		}
	}
	return null;
};

// Get a random person to query
const getRandomPerson = () => {
	return allPeople[Math.floor(Math.random() * allPeople.length)];
};

const getPartitionColor = (partitionId) => {
	return partitionColors[partitionId % partitionColors.length];
};

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

// Simulate a request with the current approach
const simulateRequest = async () => {
	if (isAnimating.value) return;

	isAnimating.value = true;
	requestPath.value = [];
	hops.value = 0;

	const person = getRandomPerson();
	const location = findPersonLocation(person);
	currentRequest.value = {
		person,
		targetNode: location.node,
		targetPartition: location.partition,
	};

	if (currentApproach.value === 1) {
		await simulateAnyNode(location);
	} else if (currentApproach.value === 2) {
		await simulateRoutingTier(location);
	} else {
		await simulateSmartClient(location);
	}

	await sleep(500);
	isAnimating.value = false;
	animationPhase.value = "idle";
};

// Approach 1: Any Node (might need forwarding)
const simulateAnyNode = async (location) => {
	// Client sends to a random node (might not be the right one)
	animationPhase.value = "sending";
	const randomNodeId = Math.floor(Math.random() * nodes.length);
	const firstNode = nodes[randomNodeId];

	requestPath.value.push({ type: "client-to-node", target: firstNode });
	highlightedNode.value = firstNode.id;
	hops.value = 1;
	await sleep(800);

	// Check if this node has the data
	if (firstNode.id !== location.node.id) {
		// Need to forward
		animationPhase.value = "forwarding";
		requestPath.value.push({
			type: "node-to-node",
			from: firstNode,
			target: location.node,
		});
		highlightedNode.value = location.node.id;
		hops.value = 2;
		await sleep(800);
	}

	// Found the data
	animationPhase.value = "responding";
	highlightedPartition.value = location.partition.id;
	await sleep(600);

	// Response back
	requestPath.value.push({ type: "response", from: location.node });
	highlightedNode.value = null;
	highlightedPartition.value = null;
};

// Approach 2: Routing Tier
const simulateRoutingTier = async (location) => {
	// Client sends to router
	animationPhase.value = "sending";
	requestPath.value.push({ type: "client-to-router" });
	hops.value = 1;
	await sleep(600);

	// Router knows the right node, forwards directly
	animationPhase.value = "routing";
	requestPath.value.push({ type: "router-to-node", target: location.node });
	highlightedNode.value = location.node.id;
	hops.value = 2;
	await sleep(800);

	// Found the data
	animationPhase.value = "responding";
	highlightedPartition.value = location.partition.id;
	await sleep(600);

	// Response back through router
	requestPath.value.push({ type: "response", from: location.node });
	highlightedNode.value = null;
	highlightedPartition.value = null;
};

// Approach 3: Smart Client
const simulateSmartClient = async (location) => {
	// Client knows exactly where to go
	animationPhase.value = "sending";
	requestPath.value.push({ type: "client-direct", target: location.node });
	highlightedNode.value = location.node.id;
	hops.value = 1;
	await sleep(800);

	// Found the data immediately
	animationPhase.value = "responding";
	highlightedPartition.value = location.partition.id;
	await sleep(600);

	// Direct response
	requestPath.value.push({ type: "response", from: location.node });
	highlightedNode.value = null;
	highlightedPartition.value = null;
};

const reset = () => {
	isAnimating.value = false;
	animationPhase.value = "idle";
	currentRequest.value = null;
	requestPath.value = [];
	highlightedNode.value = null;
	highlightedPartition.value = null;
	hops.value = 0;
};

const approachNames = {
	1: "Any Node",
	2: "Routing Tier",
	3: "Smart Client",
};

const approachDescriptions = {
	1: "Client contacts any node; node forwards if it doesn't have the data",
	2: "Client contacts partition-aware router; router forwards to correct node",
	3: "Client knows partition map; connects directly to correct node",
};

const getPartitionsForNode = (nodeId) => {
	return partitions.filter((p) => p.nodeId === nodeId);
};
</script>

<template>
  <div class="routing-demo">
    <div class="demo-header">
      <div class="approach-selector">
        <button
          v-for="n in 3"
          :key="n"
          @click="currentApproach = n; reset()"
          :class="{ active: currentApproach === n }"
          class="approach-btn"
        >
          {{ n }}. {{ approachNames[n] }}
        </button>
      </div>
      <div class="controls">
        <button @click="simulateRequest" :disabled="isAnimating" class="btn send">
          Send Request
        </button>
        <button @click="reset" class="btn reset">
          Reset
        </button>
      </div>
    </div>

    <div class="approach-description">
      {{ approachDescriptions[currentApproach] }}
    </div>

    <div class="visualization">
      <!-- Client -->
      <div class="client-section">
        <div class="client" :class="{ active: animationPhase !== 'idle' }">
          <div class="client-icon">👤</div>
          <div class="client-label">Client</div>
          <div v-if="currentRequest" class="query">
            GET "{{ currentRequest.person }}"
          </div>
        </div>
      </div>

      <!-- Connection lines -->
      <div class="connection-area">
        <!-- Approach 1: Any Node lines -->
        <template v-if="currentApproach === 1">
          <div
            class="connection-line any-node"
            :class="{ active: animationPhase === 'sending' || animationPhase === 'forwarding' }"
          >
            <span v-if="animationPhase === 'sending'" class="packet">→</span>
          </div>
          <div
            v-if="requestPath.some(r => r.type === 'node-to-node')"
            class="forward-indicator"
          >
            forward
          </div>
        </template>

        <!-- Approach 2: Routing Tier -->
        <template v-if="currentApproach === 2">
          <div class="router-container">
            <div
              class="router"
              :class="{ active: animationPhase === 'sending' || animationPhase === 'routing' }"
            >
              <div class="router-icon">⚡</div>
              <div class="router-label">Router</div>
              <div class="router-sublabel">(partition-aware)</div>
            </div>
          </div>
        </template>

        <!-- Approach 3: Smart Client -->
        <template v-if="currentApproach === 3">
          <div
            class="connection-line smart-client"
            :class="{ active: animationPhase === 'sending' }"
          >
            <span class="direct-label">direct</span>
          </div>
        </template>
      </div>

      <!-- Nodes -->
      <div class="nodes-section">
        <div
          v-for="node in nodes"
          :key="node.id"
          class="node"
          :class="{
            highlighted: highlightedNode === node.id,
            target: currentRequest?.targetNode.id === node.id
          }"
        >
          <div class="node-header">{{ node.name }}</div>
          <div class="node-partitions">
            <div
              v-for="partition in getPartitionsForNode(node.id)"
              :key="partition.id"
              class="partition"
              :class="{ highlighted: highlightedPartition === partition.id }"
              :style="{ borderColor: getPartitionColor(partition.id) }"
            >
              <div
                class="partition-header"
                :style="{ backgroundColor: getPartitionColor(partition.id) }"
              >
                P{{ partition.id }}
              </div>
              <div class="partition-people">
                <span
                  v-for="person in partition.people"
                  :key="person"
                  class="person"
                  :class="{ target: currentRequest?.person === person }"
                >
                  {{ person }}
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Request path summary -->
    <div class="request-summary" v-if="requestPath.length > 0">
      <div class="summary-header">
        <span class="hops-label">Hops: {{ hops }}</span>
        <span class="phase-label" :class="animationPhase">{{ animationPhase }}</span>
      </div>
      <div class="path-visualization">
        <span class="path-item">Client</span>
        <template v-for="(step, index) in requestPath" :key="index">
          <span class="path-arrow">→</span>
          <span v-if="step.type === 'client-to-router'" class="path-item router">Router</span>
          <span v-else-if="step.type === 'router-to-node'" class="path-item node">{{ step.target.name }}</span>
          <span v-else-if="step.type === 'client-to-node'" class="path-item node">{{ step.target.name }}</span>
          <span v-else-if="step.type === 'client-direct'" class="path-item node direct">{{ step.target.name }}</span>
          <span v-else-if="step.type === 'node-to-node'" class="path-item node forward">{{ step.target.name }}</span>
          <span v-else-if="step.type === 'response'" class="path-item response">✓ Found!</span>
        </template>
      </div>
    </div>

    <!-- Insights -->
    <div class="insights">
      <div v-if="currentApproach === 1" class="insight">
        <strong>Trade-off:</strong> Simple client, but may need extra hop if wrong node contacted
      </div>
      <div v-if="currentApproach === 2" class="insight">
        <strong>Trade-off:</strong> Always 2 hops (client→router→node), but router must stay in sync
      </div>
      <div v-if="currentApproach === 3" class="insight">
        <strong>Trade-off:</strong> Minimal hops, but client must maintain partition map
      </div>
    </div>
  </div>
</template>

<style scoped>
.routing-demo {
  font-family: 'Monaco', 'Menlo', monospace;
  padding: 1rem;
  background: #1e1e2e;
  border-radius: 8px;
  color: #cdd6f4;
}

.demo-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.approach-selector {
  display: flex;
  gap: 0.25rem;
}

.approach-btn {
  padding: 0.4rem 0.8rem;
  border: 1px solid #45475a;
  background: #313244;
  color: #cdd6f4;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.8rem;
  transition: all 0.2s;
}

.approach-btn:hover {
  background: #45475a;
}

.approach-btn.active {
  background: #89b4fa;
  color: #1e1e2e;
  border-color: #89b4fa;
  font-weight: bold;
}

.controls {
  display: flex;
  gap: 0.5rem;
}

.btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.2s;
}

.btn.send {
  background: #a6e3a1;
  color: #1e1e2e;
}

.btn.send:hover:not(:disabled) {
  background: #94e2d5;
}

.btn.send:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn.reset {
  background: #45475a;
  color: #cdd6f4;
}

.btn.reset:hover {
  background: #585b70;
}

.approach-description {
  color: #6c7086;
  font-size: 0.85rem;
  margin-bottom: 1rem;
  padding: 0.5rem;
  background: #313244;
  border-radius: 4px;
  text-align: center;
}

.visualization {
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  margin-bottom: 1rem;
  min-height: 200px;
}

.client-section {
  flex: 0 0 80px;
  display: flex;
  justify-content: center;
}

.client {
  text-align: center;
  padding: 0.75rem;
  background: #313244;
  border-radius: 8px;
  border: 2px solid #45475a;
  transition: all 0.3s;
}

.client.active {
  border-color: #89b4fa;
  box-shadow: 0 0 10px rgba(137, 180, 250, 0.3);
}

.client-icon {
  font-size: 1.5rem;
}

.client-label {
  font-size: 0.8rem;
  color: #89b4fa;
  margin-top: 0.25rem;
}

.query {
  font-size: 0.7rem;
  color: #f9e2af;
  margin-top: 0.5rem;
  padding: 0.25rem;
  background: #1e1e2e;
  border-radius: 4px;
}

.connection-area {
  flex: 0 0 80px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  position: relative;
}

.connection-line {
  width: 60px;
  height: 2px;
  background: #45475a;
  position: relative;
}

.connection-line.active {
  background: #89b4fa;
  animation: pulse 0.5s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

.packet {
  position: absolute;
  color: #a6e3a1;
  font-weight: bold;
  animation: movePacket 0.8s ease-in-out infinite;
}

@keyframes movePacket {
  0% { left: 0; }
  100% { left: 50px; }
}

.forward-indicator {
  font-size: 0.7rem;
  color: #f38ba8;
  margin-top: 0.25rem;
}

.direct-label {
  font-size: 0.65rem;
  color: #a6e3a1;
  position: absolute;
  top: -12px;
}

.router-container {
  display: flex;
  justify-content: center;
}

.router {
  text-align: center;
  padding: 0.5rem;
  background: #313244;
  border-radius: 8px;
  border: 2px solid #45475a;
  transition: all 0.3s;
}

.router.active {
  border-color: #f9e2af;
  box-shadow: 0 0 10px rgba(249, 226, 175, 0.3);
}

.router-icon {
  font-size: 1.2rem;
}

.router-label {
  font-size: 0.75rem;
  color: #f9e2af;
}

.router-sublabel {
  font-size: 0.6rem;
  color: #6c7086;
}

.nodes-section {
  flex: 1;
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.5rem;
}

.node {
  background: #313244;
  border-radius: 8px;
  padding: 0.5rem;
  border: 2px solid #45475a;
  transition: all 0.3s;
}

.node.highlighted {
  border-color: #89b4fa;
  box-shadow: 0 0 15px rgba(137, 180, 250, 0.4);
}

.node.target {
  border-color: #a6e3a1;
}

.node-header {
  font-size: 0.8rem;
  font-weight: bold;
  color: #89b4fa;
  margin-bottom: 0.5rem;
  text-align: center;
}

.node-partitions {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.partition {
  border: 1px solid;
  border-radius: 4px;
  overflow: hidden;
  transition: all 0.3s;
}

.partition.highlighted {
  box-shadow: 0 0 10px currentColor;
  transform: scale(1.02);
}

.partition-header {
  padding: 0.15rem 0.4rem;
  font-size: 0.7rem;
  font-weight: bold;
  color: #1e1e2e;
}

.partition-people {
  padding: 0.25rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.2rem;
  background: #1e1e2e;
}

.person {
  font-size: 0.6rem;
  padding: 0.1rem 0.3rem;
  background: #45475a;
  border-radius: 2px;
  color: #cdd6f4;
  transition: all 0.3s;
}

.person.target {
  background: #f9e2af;
  color: #1e1e2e;
  font-weight: bold;
}

.more {
  font-size: 0.6rem;
  color: #6c7086;
}

.request-summary {
  background: #313244;
  border-radius: 4px;
  padding: 0.75rem;
  margin-bottom: 0.75rem;
}

.summary-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.5rem;
}

.hops-label {
  font-size: 0.85rem;
  color: #f9e2af;
  font-weight: bold;
}

.phase-label {
  font-size: 0.75rem;
  padding: 0.15rem 0.5rem;
  border-radius: 4px;
  background: #45475a;
  text-transform: uppercase;
}

.phase-label.sending {
  background: #89b4fa;
  color: #1e1e2e;
}

.phase-label.forwarding {
  background: #f38ba8;
  color: #1e1e2e;
}

.phase-label.routing {
  background: #f9e2af;
  color: #1e1e2e;
}

.phase-label.responding {
  background: #a6e3a1;
  color: #1e1e2e;
}

.path-visualization {
  display: flex;
  align-items: center;
  gap: 0.3rem;
  flex-wrap: wrap;
}

.path-item {
  padding: 0.2rem 0.5rem;
  background: #45475a;
  border-radius: 4px;
  font-size: 0.75rem;
}

.path-item.router {
  background: #f9e2af;
  color: #1e1e2e;
}

.path-item.node {
  background: #89b4fa;
  color: #1e1e2e;
}

.path-item.direct {
  background: #a6e3a1;
}

.path-item.forward {
  background: #f38ba8;
}

.path-item.response {
  background: #a6e3a1;
  color: #1e1e2e;
  font-weight: bold;
}

.path-arrow {
  color: #6c7086;
}

.insights {
  border-top: 1px solid #45475a;
  padding-top: 0.75rem;
}

.insight {
  font-size: 0.8rem;
  color: #cdd6f4;
  padding: 0.5rem;
  background: #313244;
  border-radius: 4px;
  border-left: 3px solid #89b4fa;
}

.insight strong {
  color: #f9e2af;
}
</style>
