<script setup>
import { ref, computed, nextTick } from "vue";

// Each node has a fixed number of partitions (simplified to 2 per node for demo)
const partitionsPerNode = 2;
const currentNodes = ref(1);
const isAnimating = ref(false);
const animationPhase = ref("idle"); // idle, splitting, moving

// Partitions: each has id, nodeId, size (simulated data size)
const partitions = ref([
	{ id: 0, nodeId: 0, size: 50 },
	{ id: 1, nodeId: 0, size: 50 },
]);

let nextPartitionId = 2;

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
				size: p.size,
			}));

		nodes.push({
			id: n,
			partitions: nodePartitions,
			partitionCount: nodePartitions.length,
			totalSize: nodePartitions.reduce((sum, p) => sum + p.size, 0),
		});
	}
	return nodes;
});

const totalPartitionCount = computed(() => partitions.value.length);

const totalLoad = computed(() => partitions.value.reduce((sum, p) => sum + p.size, 0));

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

const addNode = async () => {
	if (isAnimating.value || currentNodes.value >= 4) return;
	isAnimating.value = true;
	animationPhase.value = "splitting";

	const newNodeId = currentNodes.value;

	// Randomly select partitions to split (one from each existing node ideally)
	// New node needs `partitionsPerNode` partitions
	const partitionsToSplit = [];
	const shuffled = [...partitions.value].sort(() => Math.random() - 0.5);

	for (let i = 0; i < partitionsPerNode && i < shuffled.length; i++) {
		partitionsToSplit.push(shuffled[i]);
	}

	await sleep(400);

	// Split selected partitions and give half to new node
	const newPartitions = [];
	const splitInfo = []; // Track which partitions were split for animation

	partitionsToSplit.forEach((partition) => {
		const newPartitionId = nextPartitionId++;
		const halfSize = Math.round(partition.size / 2);
		const remainingSize = partition.size - halfSize;

		// Reduce original partition size
		const idx = partitions.value.findIndex((p) => p.id === partition.id);
		partitions.value[idx] = {
			...partition,
			size: remainingSize,
		};

		// Create new partition for new node
		newPartitions.push({
			id: newPartitionId,
			nodeId: newNodeId,
			size: halfSize,
		});

		splitInfo.push({
			originalId: partition.id,
			newId: newPartitionId,
		});
	});

	// Capture positions before adding new node
	const firstPositions = capturePositions();

	// Add new node and new partitions
	currentNodes.value++;
	partitions.value = [...partitions.value, ...newPartitions];

	animationPhase.value = "moving";
	await nextTick();

	// Capture positions after
	const lastPositions = capturePositions();

	// FLIP animation for new partitions
	const transformUpdates = {};
	splitInfo.forEach(({ originalId, newId }) => {
		const origKey = `p-${originalId}`;
		const newKey = `p-${newId}`;
		const origFirst = firstPositions[origKey];
		const newLast = lastPositions[newKey];

		if (origFirst && newLast) {
			const deltaX = origFirst.x - newLast.x;
			const deltaY = origFirst.y - newLast.y;
			transformUpdates[newId] = {
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

	// Animate to final position
	requestAnimationFrame(() => {
		const playUpdates = {};
		splitInfo.forEach(({ newId }) => {
			playUpdates[newId] = {
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
	const partitionsToRemove = partitions.value.filter((p) => p.nodeId === removedNodeId);
	const remainingPartitions = partitions.value.filter((p) => p.nodeId !== removedNodeId);

	// Capture positions before
	const firstPositions = capturePositions();

	// Each partition from the removed node merges back with one on remaining nodes
	// The merged partition absorbs the removed partition's size
	partitionsToRemove.forEach((removedPartition, idx) => {
		// Find a partition on another node to merge with (round-robin)
		const targetIdx = idx % remainingPartitions.length;
		remainingPartitions[targetIdx] = {
			...remainingPartitions[targetIdx],
			size: remainingPartitions[targetIdx].size + removedPartition.size,
		};
	});

	partitions.value = remainingPartitions;
	currentNodes.value--;

	await nextTick();

	// Capture positions after
	const lastPositions = capturePositions();

	// Simple animation - just fade transition handled by Vue
	await sleep(600);

	animationPhase.value = "idle";
	isAnimating.value = false;
};

const reset = () => {
	if (isAnimating.value) return;
	partitions.value = [
		{ id: 0, nodeId: 0, size: 50 },
		{ id: 1, nodeId: 0, size: 50 },
	];
	nextPartitionId = 2;
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
</script>

<template>
  <div class="proportional-partitions-demo">
    <div class="demo-header">
      <div class="info">
        <span class="partitions-label">{{ totalPartitionCount }} partitions</span>
        <span class="nodes-label">{{ currentNodes }} node{{ currentNodes !== 1 ? 's' : '' }}</span>
        <span class="ratio">({{ partitionsPerNode }} per node)</span>
        <span v-if="animationPhase === 'splitting'" class="phase-label splitting">
          Splitting partitions...
        </span>
        <span v-if="animationPhase === 'moving'" class="phase-label moving">
          Rebalancing...
        </span>
      </div>
      <div class="controls">
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

    <div class="formula-info">
      <span class="formula">{{ currentNodes }} nodes × {{ partitionsPerNode }} = {{ totalPartitionCount }} partitions</span>
      <span v-if="totalLoad !== 100" class="load-warning">(total: {{ totalLoad }}%)</span>
    </div>

    <div class="nodes-grid">
      <div
        v-for="node in nodeDistribution"
        :key="node.id"
        class="node-card"
      >
        <div class="node-header">
          <span>Node {{ node.id + 1 }}</span>
          <span class="partition-count">({{ node.partitionCount }}p)</span>
        </div>
        <div class="node-content">
          <div class="partitions-list">
            <div
              v-for="partition in node.partitions"
              :key="partition.id"
              :ref="(el) => setPartitionRef(el, partition.id)"
              class="partition-box"
              :style="{
                backgroundColor: getPartitionColor(partition.id),
                ...getPartitionStyle(partition.id),
              }"
            >
              <span class="partition-id">P{{ partition.id }}</span>
              <span class="partition-size">{{ partition.size }}%</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="insight" v-if="animationPhase === 'idle'">
      <span v-if="currentNodes === 1" class="insight-text">
        Add a node to see random partition splitting
      </span>
      <span v-else class="insight-text">
        New nodes split random partitions and take half
      </span>
    </div>
  </div>
</template>

<style scoped>
.proportional-partitions-demo {
  font-family: "Monaco", "Menlo", monospace;
  padding: 0.8rem;
  background: #1e1e2e;
  border-radius: 8px;
  color: #cdd6f4;
  font-size: 0.85rem;
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
  gap: 0.8rem;
  align-items: center;
  flex-wrap: wrap;
}

.partitions-label {
  color: #cba6f7;
  font-weight: bold;
}

.nodes-label {
  color: #a6e3a1;
}

.ratio {
  color: #6c7086;
  font-size: 0.75rem;
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

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

.formula-info {
  margin-bottom: 0.8rem;
  padding: 0.4rem 0.6rem;
  background: #313244;
  border-radius: 4px;
  text-align: center;
}

.formula {
  color: #89b4fa;
  font-size: 0.8rem;
}

.load-warning {
  color: #f38ba8;
  font-size: 0.75rem;
  margin-left: 0.5rem;
}

.controls {
  display: flex;
  gap: 0.4rem;
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
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 0.5rem;
  margin-bottom: 0.8rem;
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
  background: #45475a;
  border-radius: 4px 4px 0 0;
  border-bottom: 1px solid #6c7086;
}

.partition-count {
  font-weight: normal;
  font-size: 0.65rem;
  opacity: 0.8;
}

.node-content {
  padding: 0.4rem;
  min-height: 80px;
}

.partitions-list {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
}

.partition-box {
  padding: 0.4rem 0.5rem;
  border-radius: 4px;
  color: #1e1e2e;
  font-weight: bold;
  font-size: 0.7rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: all 0.3s ease;
}

.partition-id {
  font-weight: bold;
}

.partition-size {
  font-size: 0.6rem;
  opacity: 0.8;
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
</style>
