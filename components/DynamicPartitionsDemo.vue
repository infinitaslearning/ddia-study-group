<script setup>
import { ref, computed, nextTick } from "vue";

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

// Start with only 6 people
const initialPeople = allPeople.slice(0, 6);
const nextPersonIndex = ref(6);

// Dynamic partitions - start with 1 partition containing initial data
// Each partition has: id, nodeId, people[]
const partitions = ref([
	{ id: 0, nodeId: 0, people: [...initialPeople] },
]);

let nextPartitionId = 1;
const currentNodes = ref(1);
const isAnimating = ref(false);
const animationPhase = ref("idle"); // idle, splitting, moving, adding
const splitThreshold = 8; // Split when partition has more than this many entries

const partitionRefs = ref({});
const partitionTransforms = ref({});

const getPartitionColor = (partitionId) => {
	const colors = [
		"#89b4fa",
		"#a6e3a1",
		"#f9e2af",
		"#cba6f7",
		"#f38ba8",
		"#94e2d5",
		"#fab387",
		"#74c7ec",
	];
	return colors[partitionId % colors.length];
};

const nodeDistribution = computed(() => {
	const nodeCount = currentNodes.value;
	const nodes = [];

	for (let n = 0; n < nodeCount; n++) {
		const nodePartitions = partitions.value
			.filter((p) => p.nodeId === n)
			.map((p) => ({
				id: p.id,
				people: p.people,
				size: p.people.length,
				canSplit: p.people.length > splitThreshold,
			}));

		nodes.push({
			id: n,
			partitions: nodePartitions,
			partitionCount: nodePartitions.length,
			totalEntries: nodePartitions.reduce((sum, p) => sum + p.people.length, 0),
		});
	}
	return nodes;
});

const totalPartitionCount = computed(() => partitions.value.length);

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

const capturePositions = () => {
	const positions = {};
	Object.entries(partitionRefs.value).forEach(([key, el]) => {
		if (el) {
			const rect = el.getBoundingClientRect();
			positions[key] = { x: rect.left, y: rect.top, width: rect.width, height: rect.height };
		}
	});
	return positions;
};

const canAddItem = computed(() => {
	return nextPersonIndex.value < allPeople.length;
});

const addItem = async () => {
	if (isAnimating.value || !canAddItem.value) return;
	isAnimating.value = true;
	animationPhase.value = "adding";

	// Get the next person
	const newPerson = allPeople[nextPersonIndex.value];
	nextPersonIndex.value++;

	// Find the partition with the fewest entries (or first one)
	const targetPartition = partitions.value.reduce((min, p) =>
		p.people.length < min.people.length ? p : min
	);

	// Add the person to that partition
	const partitionIndex = partitions.value.findIndex((p) => p.id === targetPartition.id);
	partitions.value[partitionIndex] = {
		...targetPartition,
		people: [...targetPartition.people, newPerson],
	};

	await nextTick();
	await sleep(300);

	// Check if we need to auto-split
	const updatedPartition = partitions.value[partitionIndex];
	if (updatedPartition.people.length > splitThreshold) {
		animationPhase.value = "splitting";
		await sleep(200);

		// Split this partition
		const midpoint = Math.ceil(updatedPartition.people.length / 2);
		const firstHalf = updatedPartition.people.slice(0, midpoint);
		const secondHalf = updatedPartition.people.slice(midpoint);

		const newPartitionId = nextPartitionId++;

		// Determine which node gets the new partition
		let targetNodeId = updatedPartition.nodeId;
		if (currentNodes.value > 1) {
			const nodeCounts = {};
			for (let i = 0; i < currentNodes.value; i++) {
				nodeCounts[i] = partitions.value.filter((p) => p.nodeId === i).length;
			}
			targetNodeId = Object.entries(nodeCounts).reduce((min, [nodeId, count]) =>
				count < nodeCounts[min] ? parseInt(nodeId) : min
			, 0);
		}

		// Update the original partition with first half
		partitions.value[partitionIndex] = {
			...updatedPartition,
			people: firstHalf,
		};

		// Add new partition with second half
		partitions.value.push({
			id: newPartitionId,
			nodeId: targetNodeId,
			people: secondHalf,
		});

		await nextTick();
		await sleep(600);
	}

	animationPhase.value = "idle";
	isAnimating.value = false;
};

const addNode = async () => {
	if (isAnimating.value || currentNodes.value >= 4) return;
	isAnimating.value = true;

	const newNodeId = currentNodes.value;

	// Calculate which partitions will move BEFORE adding the node
	const partitionsPerNodeAfter = Math.floor(partitions.value.length / (currentNodes.value + 1));
	const partitionsToMoveIds = [];

	for (let nodeId = 0; nodeId < currentNodes.value; nodeId++) {
		const nodePartitions = partitions.value.filter((p) => p.nodeId === nodeId);
		const excess = nodePartitions.length - partitionsPerNodeAfter;
		if (excess > 0 && partitionsToMoveIds.length < partitionsPerNodeAfter) {
			const toMove = nodePartitions.slice(-Math.min(excess, partitionsPerNodeAfter - partitionsToMoveIds.length));
			partitionsToMoveIds.push(...toMove.map((p) => p.id));
		}
	}

	// FLIP Step 1: Capture FIRST positions
	const firstPositions = capturePositions();

	// Add the new node
	currentNodes.value++;
	animationPhase.value = "moving";
	await nextTick();
	await sleep(400);

	// Move partitions to new node
	partitionsToMoveIds.forEach((id) => {
		const idx = partitions.value.findIndex((part) => part.id === id);
		if (idx !== -1) {
			partitions.value[idx] = { ...partitions.value[idx], nodeId: newNodeId };
		}
	});
	await nextTick();

	// FLIP Step 2: Capture LAST positions
	const lastPositions = capturePositions();

	// FLIP Step 3: INVERT - apply inverse transforms
	const transformUpdates = {};
	partitionsToMoveIds.forEach((id) => {
		const key = `p-${id}`;
		const first = firstPositions[key];
		const last = lastPositions[key];

		if (first && last) {
			const deltaX = first.x - last.x;
			const deltaY = first.y - last.y;
			transformUpdates[id] = {
				transform: `translate(${deltaX}px, ${deltaY}px)`,
				transition: "none",
			};
		}
	});

	partitionTransforms.value = { ...transformUpdates };
	await nextTick();

	// Force reflow
	Object.keys(partitionRefs.value).forEach((key) => {
		const el = partitionRefs.value[key];
		if (el) el.offsetHeight;
	});

	// FLIP Step 4: PLAY - animate to final position
	requestAnimationFrame(() => {
		const playUpdates = {};
		partitionsToMoveIds.forEach((id) => {
			playUpdates[id] = {
				transform: "translate(0, 0)",
				transition: "transform 0.6s cubic-bezier(0.4, 0, 0.2, 1)",
			};
		});
		partitionTransforms.value = { ...playUpdates };
	});

	await sleep(700);

	partitionTransforms.value = {};
	animationPhase.value = "idle";
	isAnimating.value = false;
};

const removeNode = async () => {
	if (isAnimating.value || currentNodes.value <= 1) return;
	isAnimating.value = true;
	animationPhase.value = "moving";

	const removedNodeId = currentNodes.value - 1;
	const remainingNodes = currentNodes.value - 1;

	// Calculate target nodes for partitions being moved
	const partitionsToMove = partitions.value.filter((p) => p.nodeId === removedNodeId);
	const moveMap = {}; // partitionId -> targetNodeId
	partitionsToMove.forEach((p, idx) => {
		moveMap[p.id] = idx % remainingNodes;
	});

	// FLIP Step 1: Capture FIRST positions
	const firstPositions = capturePositions();

	// Move partitions and remove node
	Object.entries(moveMap).forEach(([id, targetNode]) => {
		const partIdx = partitions.value.findIndex((part) => part.id === parseInt(id, 10));
		if (partIdx !== -1) {
			partitions.value[partIdx] = { ...partitions.value[partIdx], nodeId: targetNode };
		}
	});
	currentNodes.value--;
	await nextTick();

	// FLIP Step 2: Capture LAST positions
	const lastPositions = capturePositions();

	// FLIP Step 3: INVERT
	const transformUpdates = {};
	Object.keys(moveMap).forEach((id) => {
		const key = `p-${id}`;
		const first = firstPositions[key];
		const last = lastPositions[key];

		if (first && last) {
			const deltaX = first.x - last.x;
			const deltaY = first.y - last.y;
			transformUpdates[id] = {
				transform: `translate(${deltaX}px, ${deltaY}px)`,
				transition: "none",
			};
		}
	});

	partitionTransforms.value = { ...transformUpdates };
	await nextTick();

	// Force reflow
	Object.keys(partitionRefs.value).forEach((key) => {
		const el = partitionRefs.value[key];
		if (el) el.offsetHeight;
	});

	// FLIP Step 4: PLAY
	requestAnimationFrame(() => {
		const playUpdates = {};
		Object.keys(moveMap).forEach((id) => {
			playUpdates[id] = {
				transform: "translate(0, 0)",
				transition: "transform 0.6s cubic-bezier(0.4, 0, 0.2, 1)",
			};
		});
		partitionTransforms.value = { ...playUpdates };
	});

	await sleep(700);

	partitionTransforms.value = {};
	animationPhase.value = "idle";
	isAnimating.value = false;
};

const reset = () => {
	if (isAnimating.value) return;
	partitions.value = [{ id: 0, nodeId: 0, people: [...initialPeople] }];
	nextPartitionId = 1;
	nextPersonIndex.value = 6;
	currentNodes.value = 1;
	animationPhase.value = "idle";
	partitionTransforms.value = {};
};

const setPartitionRef = (el, partitionId) => {
	if (el) partitionRefs.value[`p-${partitionId}`] = el;
};

const getPartitionStyle = (partitionId) => {
	const transform = partitionTransforms.value[partitionId];
	if (transform) {
		return {
			transform: transform.transform,
			transition: transform.transition,
			zIndex: 100,
		};
	}
	return {};
};

const getSizeIndicator = (size) => {
	if (size > splitThreshold) return "too-large";
	return "ok";
};
</script>

<template>
  <div class="dynamic-partitions-demo">
    <div class="demo-header">
      <div class="info">
        <span class="dynamic-label">Dynamic: {{ totalPartitionCount }} partition{{ totalPartitionCount !== 1 ? 's' : '' }}</span>
        <span class="nodes-label">{{ currentNodes }} node{{ currentNodes !== 1 ? 's' : '' }}</span>
        <span v-if="animationPhase === 'splitting'" class="phase-label splitting">
          Splitting partition...
        </span>
        <span v-if="animationPhase === 'moving'" class="phase-label moving">
          Rebalancing...
        </span>
        <span v-if="animationPhase === 'adding'" class="phase-label adding">
          Adding entry...
        </span>
      </div>
      <div class="controls">
        <button @click="addItem" :disabled="isAnimating || !canAddItem" class="btn add-entry">
          + Entry
        </button>
        <button @click="addNode" :disabled="isAnimating || currentNodes >= 4" class="btn add">
          + Node
        </button>
        <button @click="removeNode" :disabled="isAnimating || currentNodes <= 1" class="btn remove">
          − Node
        </button>
        <button @click="reset" :disabled="isAnimating" class="btn reset">
          Reset
        </button>
      </div>
    </div>

    <div class="threshold-info">
      <span class="threshold">Split when &gt;{{ splitThreshold }} entries</span>
    </div>

    <div class="nodes-grid">
      <div
        v-for="node in nodeDistribution"
        :key="node.id"
        class="node-card"
      >
        <div class="node-header">
          <span>Node {{ node.id + 1 }}</span>
          <span class="partition-count">({{ node.partitionCount }}p, {{ node.totalEntries }} entries)</span>
        </div>
        <div class="node-content">
          <div v-if="node.partitions.length === 0" class="empty-placeholder">
            no partitions
          </div>
          <div class="partitions-list">
            <div
              v-for="partition in node.partitions"
              :key="partition.id"
              :ref="(el) => setPartitionRef(el, partition.id)"
              class="partition-box"
              :class="[getSizeIndicator(partition.size)]"
              :style="{
                borderColor: getPartitionColor(partition.id),
                ...getPartitionStyle(partition.id),
              }"
            >
              <div class="partition-header" :style="{ backgroundColor: getPartitionColor(partition.id) }">
                <span>P{{ partition.id }}</span>
                <span class="size-badge">{{ partition.size }}</span>
              </div>
              <div class="partition-people">
                <span v-for="person in partition.people" :key="person" class="person">
                  {{ person }}
                </span>
              </div>
              <div v-if="partition.canSplit" class="action-hint split-hint">can split</div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="insight" v-if="animationPhase === 'idle'">
      <span v-if="totalPartitionCount === 1" class="insight-text warning">
        ⚠ Single partition = no parallelism (cold start problem)
      </span>
      <span v-else class="insight-text">
        Partitions adapt to data size automatically
      </span>
    </div>
  </div>
</template>

<style scoped>
.dynamic-partitions-demo {
  font-family: "Monaco", "Menlo", monospace;
  padding: 0.8rem;
  background: #1e1e2e;
  border-radius: 8px;
  color: #cdd6f4;
  font-size: 0.85rem;
  position: relative;
}

.demo-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.info {
  display: flex;
  gap: 1rem;
  align-items: center;
  flex-wrap: wrap;
}

.dynamic-label {
  color: #cba6f7;
  font-weight: bold;
}

.nodes-label {
  color: #a6e3a1;
}

.phase-label {
  padding: 0.2rem 0.6rem;
  border-radius: 4px;
  font-size: 0.75rem;
  animation: pulse 1s infinite;
}

.phase-label.splitting {
  background: #f9e2af;
  color: #1e1e2e;
}

.phase-label.moving {
  background: #89b4fa;
  color: #1e1e2e;
}

.phase-label.adding {
  background: #cba6f7;
  color: #1e1e2e;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

.threshold-info {
  display: flex;
  gap: 1rem;
  margin-bottom: 0.8rem;
  font-size: 0.7rem;
  color: #6c7086;
}

.threshold {
  background: #313244;
  padding: 0.2rem 0.5rem;
  border-radius: 3px;
}

.controls {
  display: flex;
  gap: 0.4rem;
  flex-wrap: wrap;
}

.btn {
  padding: 0.4rem 0.6rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  font-size: 0.75rem;
  transition: all 0.2s;
}

.btn.add-entry {
  background: #cba6f7;
  color: #1e1e2e;
}

.btn.add {
  background: #a6e3a1;
  color: #1e1e2e;
}

.btn.remove {
  background: #f38ba8;
  color: #1e1e2e;
}

.btn.reset {
  background: #45475a;
  color: #cdd6f4;
}

.btn:hover:not(:disabled) {
  opacity: 0.9;
  transform: scale(1.02);
}

.btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.nodes-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.node-card {
  background: #313244;
  border-radius: 6px;
  border: 2px solid #6c7086;
  overflow: visible;
}

.node-header {
  padding: 0.25rem 0.4rem;
  color: #cdd6f4;
  font-weight: bold;
  font-size: 0.75rem;
  text-align: center;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 0.3rem;
  border-radius: 4px 4px 0 0;
  background: #45475a;
  border-bottom: 1px solid #6c7086;
}

.partition-count {
  font-weight: normal;
  font-size: 0.65rem;
  opacity: 0.8;
}

.node-content {
  padding: 0.4rem;
  min-height: 100px;
  position: relative;
}

.empty-placeholder {
  color: #6c7086;
  font-size: 0.65rem;
  font-style: italic;
  text-align: center;
  padding: 1rem 0.5rem;
}

.partitions-list {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.partition-box {
  background: #1e1e2e;
  border: 2px solid;
  border-radius: 4px;
  overflow: hidden;
  position: relative;
  transition: all 0.3s ease;
}

.partition-box.too-large {
  box-shadow: 0 0 8px rgba(249, 226, 175, 0.5);
}

.partition-header {
  padding: 0.15rem 0.3rem;
  font-size: 0.6rem;
  font-weight: bold;
  color: #1e1e2e;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.3rem;
}

.size-badge {
  background: rgba(0, 0, 0, 0.2);
  padding: 0.05rem 0.2rem;
  border-radius: 2px;
  font-size: 0.55rem;
}

.partition-people {
  padding: 0.25rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.2rem;
  max-height: 80px;
  overflow-y: auto;
}

.person {
  background: #45475a;
  padding: 0.1rem 0.3rem;
  border-radius: 3px;
  font-size: 0.55rem;
  color: #cdd6f4;
}

.action-hint {
  position: absolute;
  bottom: 2px;
  right: 4px;
  font-size: 0.5rem;
  padding: 0.1rem 0.2rem;
  border-radius: 2px;
  opacity: 0.8;
}

.split-hint {
  background: #f9e2af;
  color: #1e1e2e;
}

.insight {
  background: #313244;
  border-radius: 6px;
  padding: 0.4rem;
  text-align: center;
}

.insight-text {
  color: #6c7086;
  font-size: 0.75rem;
}

.insight-text.warning {
  color: #f9e2af;
}
</style>
