<script setup>
import { ref, computed } from "vue";

const hashValue = 123456;
const currentN = ref(10);
const history = ref([]);
const isAnimating = ref(false);

const currentPartition = computed(() => hashValue % currentN.value);

const addNode = async () => {
	if (isAnimating.value) return;
	isAnimating.value = true;

	const oldN = currentN.value;
	const oldPartition = hashValue % oldN;

	currentN.value++;

	const newPartition = hashValue % currentN.value;

	history.value.push({
		n: currentN.value,
		partition: newPartition,
		moved: oldPartition !== newPartition,
		from: oldPartition,
	});

	setTimeout(() => {
		isAnimating.value = false;
	}, 500);
};

const reset = () => {
	currentN.value = 10;
	history.value = [];
};

const getPartitionColor = (partition) => {
	const colors = [
		"#ef4444",
		"#f97316",
		"#eab308",
		"#22c55e",
		"#14b8a6",
		"#06b6d4",
		"#3b82f6",
		"#8b5cf6",
		"#d946ef",
		"#ec4899",
		"#f43f5e",
		"#84cc16",
	];
	return colors[partition % colors.length];
};
</script>

<template>
  <div class="hash-mod-demo">
    <div class="demo-header">
      <div class="hash-display">
        <span class="label">hash(key) = </span>
        <span class="value">{{ hashValue }}</span>
      </div>
      <div class="controls">
        <button @click="addNode" :disabled="isAnimating || currentN >= 20" class="btn add">
          + Add Partition
        </button>
        <button @click="reset" class="btn reset">
          Reset
        </button>
      </div>
    </div>

    <div class="current-state">
      <div class="formula">
        <span>{{ hashValue }}</span>
        <span class="op">%</span>
        <span class="n">{{ currentN }}</span>
        <span class="op">=</span>
        <span
          class="result"
          :style="{ backgroundColor: getPartitionColor(currentPartition) }"
        >
          Partition {{ currentPartition }}
        </span>
      </div>
      <div class="nodes-label">{{ currentN }} partitions in cluster</div>
    </div>

    <div class="history" v-if="history.length > 0">
      <div class="history-title">Migration History:</div>
      <div class="history-items">
        <div
          v-for="(item, index) in history"
          :key="index"
          class="history-item"
          :class="{ moved: item.moved }"
        >
          <span class="n-change">N={{ item.n }}</span>
          <span v-if="item.moved" class="migration">
            <span
              class="from-partition"
              :style="{ backgroundColor: getPartitionColor(item.from) }"
            >P{{ item.from }}</span>
            <span class="arrow"> → </span>
            <span
              class="to-partition"
              :style="{ backgroundColor: getPartitionColor(item.partition) }"
            >P{{ item.partition }}</span>
            <span class="moved-label"> MOVED!</span>
          </span>
          <span v-else class="no-move">
            <span
              class="same-partition"
              :style="{ backgroundColor: getPartitionColor(item.partition) }"
            >P{{ item.partition }}</span>
            <span class="stayed-label"> stayed</span>
          </span>
        </div>
      </div>
    </div>

    <div class="insight" v-if="history.length >= 3">
      <div class="insight-text">
        {{ history.filter(h => h.moved).length }} / {{ history.length }} additions caused migration
      </div>
    </div>
  </div>
</template>

<style scoped>
.hash-mod-demo {
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
  margin-bottom: 1.5rem;
}

.hash-display {
  font-size: 1.1rem;
}

.hash-display .label {
  color: #89b4fa;
}

.hash-display .value {
  color: #f9e2af;
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

.btn.add {
  background: #a6e3a1;
  color: #1e1e2e;
}

.btn.add:hover:not(:disabled) {
  background: #94e2d5;
}

.btn.add:disabled {
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

.current-state {
  text-align: center;
  margin-bottom: 1.5rem;
}

.formula {
  font-size: 1.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.formula .op {
  color: #89b4fa;
}

.formula .n {
  color: #a6e3a1;
  font-weight: bold;
}

.formula .result {
  padding: 0.3rem 0.8rem;
  border-radius: 4px;
  color: #1e1e2e;
  font-weight: bold;
}

.nodes-label {
  margin-top: 0.5rem;
  color: #6c7086;
  font-size: 0.9rem;
}

.history {
  border-top: 1px solid #45475a;
  padding-top: 1rem;
}

.history-title {
  color: #89b4fa;
  margin-bottom: 0.5rem;
  font-size: 0.9rem;
}

.history-items {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.history-item {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.3rem 0.6rem;
  background: #313244;
  border-radius: 4px;
  font-size: 0.85rem;
}

.history-item.moved {
  border-left: 3px solid #f38ba8;
}

.n-change {
  color: #a6e3a1;
}

.from-partition, .to-partition, .same-partition {
  padding: 0.1rem 0.4rem;
  border-radius: 3px;
  color: #1e1e2e;
  font-weight: bold;
  font-size: 0.8rem;
}

.arrow {
  color: #f38ba8;
}

.moved-label {
  color: #f38ba8;
  font-weight: bold;
  font-size: 0.75rem;
}

.stayed-label {
  color: #6c7086;
  font-size: 0.75rem;
}

.insight {
  margin-top: 1rem;
  padding: 0.8rem;
  background: #45475a;
  border-radius: 4px;
  text-align: center;
}

.insight-text {
  color: #f38ba8;
  font-weight: bold;
}
</style>
