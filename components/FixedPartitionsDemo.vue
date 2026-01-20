<script setup>
import { ref, computed, nextTick } from "vue";

const people = [
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

const totalPartitions = 6;
const currentNodes = ref(2);
const isAnimating = ref(false);
const animationPhase = ref("idle"); // idle, newNode, moving, complete
const migratingPartitions = ref([]);

// Track partition ownership
const partitionOwnership = ref(
	Array.from({ length: totalPartitions }, (_, i) => Math.floor(i / (totalPartitions / 2)))
);

// For FLIP animation - store transform values
const partitionTransforms = ref({});

// Refs for measuring positions
const containerRef = ref(null);
const partitionRefs = ref({});

const personToPartition = computed(() => {
	const map = {};
	people.forEach((person, idx) => {
		map[person] = idx % totalPartitions;
	});
	return map;
});

const getPeopleInPartition = (partitionId) => {
	return people.filter((p) => personToPartition.value[p] === partitionId);
};

const getPartitionsForNode = (nodeId) => {
	return partitionOwnership.value
		.map((owner, partitionId) => (owner === nodeId ? partitionId : -1))
		.filter((p) => p !== -1);
};

const calculateStealPartitions = (newNodeId, nodeCount) => {
	const basePerNode = Math.floor(totalPartitions / nodeCount);
	const remainder = totalPartitions % nodeCount;
	const partitionsToSteal = [];

	// Calculate target for new node: it gets basePerNode,
	// plus 1 extra if newNodeId < remainder
	const targetForNewNode = basePerNode + (newNodeId < remainder ? 1 : 0);

	// Steal from nodes that have more than their target
	for (let existingNode = 0; existingNode < nodeCount - 1; existingNode++) {
		const currentPartitions = getPartitionsForNode(existingNode);
		// Target for this existing node
		const targetForThisNode = basePerNode + (existingNode < remainder ? 1 : 0);
		const toSteal = currentPartitions.length - targetForThisNode;

		if (toSteal > 0 && partitionsToSteal.length < targetForNewNode) {
			const stealCount = Math.min(toSteal, targetForNewNode - partitionsToSteal.length);
			for (let i = 0; i < stealCount; i++) {
				const partitionToSteal = currentPartitions[currentPartitions.length - 1 - i];
				partitionsToSteal.push({
					partition: partitionToSteal,
					fromNode: existingNode,
					toNode: newNodeId,
					people: getPeopleInPartition(partitionToSteal),
				});
			}
		}
	}

	return partitionsToSteal;
};

const calculateRedistributePartitions = (removedNodeId, newNodeCount) => {
	const partitionsToMove = [];
	const removedPartitions = getPartitionsForNode(removedNodeId);

	// Calculate target distribution after removal
	const basePerNode = Math.floor(totalPartitions / newNodeCount);
	const remainder = totalPartitions % newNodeCount;

	// Find nodes that need more partitions to reach their target
	const nodesNeedingPartitions = [];
	for (let nodeId = 0; nodeId < newNodeCount; nodeId++) {
		const currentCount = getPartitionsForNode(nodeId).length;
		const targetCount = basePerNode + (nodeId < remainder ? 1 : 0);
		const needed = targetCount - currentCount;
		for (let i = 0; i < needed; i++) {
			nodesNeedingPartitions.push(nodeId);
		}
	}

	// Distribute removed partitions to nodes that need them
	removedPartitions.forEach((partition, idx) => {
		const targetNode = nodesNeedingPartitions[idx] ?? (idx % newNodeCount);
		partitionsToMove.push({
			partition,
			fromNode: removedNodeId,
			toNode: targetNode,
			people: getPeopleInPartition(partition),
		});
	});

	return partitionsToMove;
};

const getPartitionColor = (partitionId) => {
	const colors = [
		"#89b4fa",
		"#a6e3a1",
		"#f9e2af",
		"#cba6f7",
		"#f38ba8",
		"#94e2d5",
	];
	return colors[partitionId % colors.length];
};

const nodeDistribution = computed(() => {
	const nodeCount = currentNodes.value;
	const nodes = [];

	for (let n = 0; n < nodeCount; n++) {
		const partitionIds = getPartitionsForNode(n);
		const partitionsData = partitionIds.map((p) => ({
			id: p,
			people: getPeopleInPartition(p),
			isMigrating: migratingPartitions.value.some((m) => m.partition === p),
		}));

		nodes.push({
			id: n,
			partitions: partitionsData,
			partitionCount: partitionIds.length,
			isNew: animationPhase.value === "newNode" && n === nodeCount - 1,
		});
	}
	return nodes;
});

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

// Capture current positions of all partitions
const capturePositions = () => {
	const positions = {};
	Object.entries(partitionRefs.value).forEach(([key, el]) => {
		if (el) {
			const rect = el.getBoundingClientRect();
			positions[key] = { x: rect.left, y: rect.top };
		}
	});
	return positions;
};

const addNode = async () => {
	if (isAnimating.value || currentNodes.value >= 8) return;
	isAnimating.value = true;

	const newNodeId = currentNodes.value;
	const newNodeCount = currentNodes.value + 1;

	// Calculate which partitions will move
	const migrations = calculateStealPartitions(newNodeId, newNodeCount);
	migratingPartitions.value = migrations;

	// FLIP Step 1: Capture FIRST positions (before any DOM changes)
	const firstPositions = capturePositions();

	// Phase 1: Add new empty node (but don't move partitions yet)
	currentNodes.value = newNodeCount;
	animationPhase.value = "newNode";
	await nextTick();
	await sleep(600);

	// Phase 2: Update ownership (this causes partitions to move in DOM)
	animationPhase.value = "moving";

	// Pre-apply inverse transforms BEFORE ownership changes
	// This way when DOM updates, elements will already have their transform set
	migrations.forEach((m) => {
		partitionTransforms.value[m.partition] = {
			transform: "translate(0, 0)",
			transition: "none",
		};
	});

	// Now update ownership
	migrations.forEach((m) => {
		partitionOwnership.value[m.partition] = m.toNode;
	});
	await nextTick();

	// FLIP Step 2: Capture LAST positions (after DOM update)
	const lastPositions = capturePositions();

	// FLIP Step 3: INVERT - Calculate and apply inverse transforms
	// Elements are now in their new positions, we transform them back to old positions
	const transformUpdates = {};
	migrations.forEach((m) => {
		const key = `p-${m.partition}`;
		const first = firstPositions[key];
		const last = lastPositions[key];

		if (first && last) {
			const deltaX = first.x - last.x;
			const deltaY = first.y - last.y;

			transformUpdates[m.partition] = {
				transform: `translate(${deltaX}px, ${deltaY}px)`,
				transition: "none",
			};
		}
	});

	// Apply all transforms at once
	partitionTransforms.value = { ...transformUpdates };
	await nextTick();

	// Force a reflow to ensure transforms are applied before transitioning
	Object.keys(partitionRefs.value).forEach((key) => {
		const el = partitionRefs.value[key];
		if (el) el.offsetHeight; // Force reflow
	});

	// FLIP Step 4: PLAY - Animate transforms to 0
	requestAnimationFrame(() => {
		const playUpdates = {};
		migrations.forEach((m) => {
			playUpdates[m.partition] = {
				transform: "translate(0, 0)",
				transition: "transform 0.6s cubic-bezier(0.4, 0, 0.2, 1)",
			};
		});
		partitionTransforms.value = { ...playUpdates };
	});

	await sleep(700);

	// Phase 3: Complete
	animationPhase.value = "complete";
	partitionTransforms.value = {};
	migratingPartitions.value = [];

	await sleep(200);

	animationPhase.value = "idle";
	isAnimating.value = false;
};

const removeNode = async () => {
	if (isAnimating.value || currentNodes.value <= 2) return;
	isAnimating.value = true;

	const removedNodeId = currentNodes.value - 1;
	const newNodeCount = currentNodes.value - 1;

	// Calculate which partitions will move
	const migrations = calculateRedistributePartitions(removedNodeId, newNodeCount);
	migratingPartitions.value = migrations;

	// FLIP Step 1: Capture FIRST positions
	const firstPositions = capturePositions();

	// Update ownership and remove node
	animationPhase.value = "moving";
	migrations.forEach((m) => {
		partitionOwnership.value[m.partition] = m.toNode;
	});
	currentNodes.value = newNodeCount;
	await nextTick();

	// FLIP Step 2: Capture LAST positions
	const lastPositions = capturePositions();

	// FLIP Step 3: INVERT
	const transformUpdates = {};
	migrations.forEach((m) => {
		const key = `p-${m.partition}`;
		const first = firstPositions[key];
		const last = lastPositions[key];

		if (first && last) {
			const deltaX = first.x - last.x;
			const deltaY = first.y - last.y;

			transformUpdates[m.partition] = {
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
		migrations.forEach((m) => {
			playUpdates[m.partition] = {
				transform: "translate(0, 0)",
				transition: "transform 0.6s cubic-bezier(0.4, 0, 0.2, 1)",
			};
		});
		partitionTransforms.value = { ...playUpdates };
	});

	await sleep(700);

	// Complete
	animationPhase.value = "complete";
	partitionTransforms.value = {};
	migratingPartitions.value = [];

	await sleep(200);

	animationPhase.value = "idle";
	isAnimating.value = false;
};

const reset = () => {
	if (isAnimating.value) return;
	currentNodes.value = 2;
	migratingPartitions.value = [];
	partitionTransforms.value = {};
	animationPhase.value = "idle";
	partitionOwnership.value = Array.from(
		{ length: totalPartitions },
		(_, i) => Math.floor(i / (totalPartitions / 2))
	);
};

const getNodeColor = () => {
	return "#6c7086"; // Subtle gray border for all nodes
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
  <div ref="containerRef" class="fixed-partitions-demo">
    <div class="demo-header">
      <div class="info">
        <span class="fixed-label">Fixed: {{ totalPartitions }} partitions</span>
        <span class="nodes-label">{{ currentNodes }} nodes</span>
        <span v-if="animationPhase === 'newNode'" class="phase-label new-node-phase">
          New node joining...
        </span>
        <span v-if="animationPhase === 'moving'" class="phase-label moving">
          Stealing partitions...
        </span>
      </div>
      <div class="controls">
        <button @click="removeNode" :disabled="isAnimating || currentNodes <= 2" class="btn remove">
          − Remove
        </button>
        <button @click="addNode" :disabled="isAnimating || currentNodes >= 8" class="btn add">
          + Add Node
        </button>
        <button @click="reset" :disabled="isAnimating" class="btn reset">
          Reset
        </button>
      </div>
    </div>

    <div class="nodes-grid">
      <div
        v-for="node in nodeDistribution"
        :key="node.id"
        class="node-card"
        :class="{ 'new-node': node.isNew }"
        :style="{ borderColor: getNodeColor(node.id) }"
      >
        <div class="node-header">
          <span>Node {{ node.id + 1 }}</span>
          <span class="partition-count">({{ node.partitionCount }}p)</span>
          <span v-if="node.isNew" class="new-badge">NEW</span>
        </div>
        <div class="node-content">
          <div v-if="node.partitions.length === 0 && animationPhase === 'newNode'" class="empty-placeholder">
            waiting for partitions...
          </div>
          <div class="partitions-list">
            <div
              v-for="partition in node.partitions"
              :key="partition.id"
              :ref="(el) => setPartitionRef(el, partition.id)"
              class="partition-box"
              :class="{ migrating: partition.isMigrating }"
              :style="{
                borderColor: getPartitionColor(partition.id),
                ...getPartitionStyle(partition.id),
              }"
            >
              <div class="partition-header" :style="{ backgroundColor: getPartitionColor(partition.id) }">
                P{{ partition.id }}
              </div>
              <div class="partition-people">
                <span v-for="person in partition.people" :key="person" class="person">
                  {{ person }}
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="migration-summary" v-if="migratingPartitions.length > 0 && animationPhase !== 'idle'">
      <div class="summary-content">
        <span class="summary-count">{{ migratingPartitions.length }}</span>
        <span class="summary-text">partitions moving ({{ Math.round(migratingPartitions.length / totalPartitions * 100) }}%)</span>
      </div>
    </div>

    <div class="insight" v-if="animationPhase === 'idle'">
      <span class="insight-text">Each node has ~{{ Math.round(totalPartitions / currentNodes) }} partitions</span>
    </div>
  </div>
</template>

<style scoped>
.fixed-partitions-demo {
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
  margin-bottom: 1rem;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.info {
  display: flex;
  gap: 1rem;
  align-items: center;
  flex-wrap: wrap;
}

.fixed-label {
  color: #f9e2af;
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

.phase-label.new-node-phase {
  background: #a6e3a1;
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

.controls {
  display: flex;
  gap: 0.4rem;
}

.btn {
  padding: 0.4rem 0.8rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  font-size: 0.8rem;
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
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.node-card {
  background: #313244;
  border-radius: 6px;
  border: 2px solid;
  overflow: visible;
}

.node-card.new-node {
  animation: nodeAppear 0.5s ease-out;
}

@keyframes nodeAppear {
  0% {
    opacity: 0;
    transform: scale(0.8);
  }
  100% {
    opacity: 1;
    transform: scale(1);
  }
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

.new-badge {
  background: #a6e3a1;
  color: #1e1e2e;
  padding: 0.1rem 0.25rem;
  border-radius: 3px;
  font-size: 0.55rem;
}

.node-content {
  padding: 0.4rem;
  min-height: 90px;
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
}

.partition-box.migrating {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
}

.partition-header {
  padding: 0.15rem 0.3rem;
  font-size: 0.6rem;
  font-weight: bold;
  color: #1e1e2e;
  display: flex;
  align-items: center;
  gap: 0.3rem;
}

.partition-people {
  padding: 0.25rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.2rem;
}

.person {
  background: #45475a;
  padding: 0.1rem 0.3rem;
  border-radius: 3px;
  font-size: 0.6rem;
  color: #cdd6f4;
}

.migration-summary {
  background: #45475a;
  border-radius: 6px;
  padding: 0.5rem;
  text-align: center;
  margin-bottom: 0.5rem;
}

.summary-content {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.4rem;
  flex-wrap: wrap;
}

.summary-count {
  background: #a6e3a1;
  color: #1e1e2e;
  padding: 0.15rem 0.4rem;
  border-radius: 4px;
  font-weight: bold;
  font-size: 0.9rem;
}

.summary-text {
  color: #cdd6f4;
  font-size: 0.8rem;
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
