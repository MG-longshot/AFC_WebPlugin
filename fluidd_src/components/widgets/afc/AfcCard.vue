<template>
  <div class="afc-panel-wrapper">
    <collapsable-card
      icon="$filament"
      :title="'AFC Spools'"
      draggable
      layout-path="dashboard.afc-card"
    >
      <template #buttons>
        <app-btn icon tile :title="'Refresh AFC Spools'" @click="fetchSpoolData">
          <span class="icon-refresh"></span>
        </app-btn>
      </template>
      <div v-if="totalPages > 1" class="pagination-controls">
        <app-btn icon @click="prevPage" :disabled="currentPage === 0">
          <span class="icon-chevron-left"></span>
        </app-btn>
        <span>Page {{ currentPage + 1 }} of {{ totalPages }}</span>
        <v-btn icon @click="nextPage" :disabled="currentPage >= totalPages - 1">
          <span class="icon-chevron-right"></span>
        </v-btn>
      </div>
      <div class="spool-container">
        <div v-for="(spool, index) in paginatedSpoolData" :key="index" class="spool-card">
          <div class="filament-reel" style="padding: 1rem">
            <svg width="50" height="50" viewBox="0 0 487.04 487.04">
              <g>
                <circle
                  :style="{ fill: spool.color ? `#${spool.color}` : '#999' }"
                  cx="243.52"
                  cy="243.52"
                  r="232.97"
                />
                <circle :style="{ fill: '#e6e6e6' }" cx="243.52" cy="243.52" r="112.5" />
                <path
                  :style="{ fill: '#444444' }"
                  d="M0,243.52c0,134.42,109.1,243.52,243.52,243.52,134.42,0,243.52-109.1,243.52-243.52S377.95,0,243.52,0C109.1,0,0,109.1,0,243.52Zm115.73,181.78c-52.4-39.5-86.52-98.59-94.52-163.72v-.09c-.68-5.43,1-10.89,4.6-15,3.6-4.12,8.79-6.51,14.26-6.57l118.36-1.33c18.99-.21,36.63,9.83,46.12,26.29,9.5,16.45,9.38,36.74-.3,53.09l-60.29,101.76c-2.8,4.73-7.48,8.03-12.87,9.1-5.39,1.06-10.98-.22-15.36-3.52ZM450.22,238.8c5.49,.06,10.7,2.46,14.31,6.59,3.62,4.13,5.3,9.61,4.63,15.06-8.01,65.13-42.12,124.22-94.52,163.72l-.07,.05c-4.37,3.29-9.93,4.57-15.3,3.51-5.37-1.06-10.03-4.36-12.82-9.06l-60.33-101.84c-9.68-16.34-9.8-36.64-.3-53.09,9.5-16.45,27.13-26.5,46.12-26.29l118.27,1.33ZM338.12,40.02c5.04,2.14,8.92,6.32,10.69,11.49,1.77,5.18,1.24,10.86-1.44,15.63l-58.03,103.17c-9.31,16.56-26.83,26.8-45.83,26.8-19,0-36.51-10.25-45.83-26.8l-57.99-103.09c-2.69-4.79-3.22-10.49-1.45-15.69,1.77-5.2,5.68-9.4,10.73-11.54,60.41-25.63,128.64-25.63,189.05,0l.08,.04Z"
                />
              </g>
            </svg>
          </div>
          <h3>Spool {{ index + 1 + currentPage * spoolsPerPage }}</h3>
          <p><strong>Material:</strong> {{ spool.material }}</p>
          <p>
            <strong>Remaining Weight:</strong>
            {{ spool.remaining_weight ? spool.remaining_weight.toFixed(2) : "N/A" }} g
          </p>
          <p><strong>Status:</strong> {{ determineStatus(spool) }}</p>
        </div>
      </div>
    </collapsable-card>
  </div>
</template>

<script lang="ts">
import { Component, Mixins } from "vue-property-decorator";
import StateMixin from "@/mixins/state";
import StatusLabel from "@/components/widgets/status/StatusLabel.vue";
import type { Macro } from "@/store/macros/types";
@Component({
  components: {
    StatusLabel,
  },
})
export default class AfcCard extends Mixins(StateMixin) {
  spoolData: any[] = [];
  private intervalId: ReturnType<typeof setInterval> | undefined;
  systemData: any = null;
  isPanelOpen: number[] = [0];
  currentPage: number = 0;
  spoolsPerPage: number = 4;

  async mounted() {
    await this.fetchSpoolData();
    this.intervalId = setInterval(() => {
      this.fetchSpoolData();
    }, 10000); // Refresh data every 10 seconds
  }

  beforeDestroy() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
  }

  private async fetchSpoolData() {
    try {
      const response = await fetch("/server/afc/spools");
      const data = await response.json();
      if (data.result && data.result.status === "success" && data.result.spools) {
        this.spoolData = this.extractLaneData(data.result.spools);
        this.systemData = data.result.spools.system;
      }
    } catch (error) {
      console.error("Error fetching AFC spool data:", error);
    }
  }

  private extractLaneData(spools: any) {
    const lanes = [];
    const numUnits = spools.system.num_units;
    const numLanes = spools.system.num_lanes;
    for (let i = 1; i <= numUnits; i++) {
      const unitName = `Turtle_${i}`;
      if (spools[unitName]) {
        for (let j = 1; j <= numLanes; j++) {
          const laneName = `leg${j}`;
          if (spools[unitName][laneName]) {
            lanes.push(spools[unitName][laneName]);
          }
        }
      }
    }
    return lanes;
  }

  private get totalPages() {
    return Math.ceil(this.spoolData.length / this.spoolsPerPage);
  }

  private get paginatedSpoolData() {
    const start = this.currentPage * this.spoolsPerPage;
    const end = start + this.spoolsPerPage;
    return this.spoolData.slice(start, end);
  }

  private prevPage() {
    if (this.currentPage > 0) {
      this.currentPage--;
    }
  }

  private nextPage() {
    if (this.currentPage < this.totalPages - 1) {
      this.currentPage++;
    }
  }

  private determineStatus(spool: any) {
    if (spool.load && spool.prep && spool.loaded_to_hub) {
      if (this.systemData && this.systemData.current_load === `leg${spool.LANE}`) {
        return "Locked and loaded to tool";
      }
      return "Locked and Loaded";
    }
    return "Not Loaded";
  }
}
</script>

<style scoped>
.afc-panel {
  background-color: #1e1e1e;
  color: #ffffff;
  margin-bottom: 16px;
}

.afc-panel-wrapper {
  margin-bottom: 24px;
}

.v-card-title {
  font-weight: bold;
  font-size: 1.5em;
  text-align: center;
  padding-bottom: 16px;
}

.spool-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-around;
  gap: 16px;
  padding: 16px;
}

.spool-card {
  background-color: #2e2e2e;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  max-width: 300px;
  width: 100%;
  position: relative;
}

.filament-reel {
  position: absolute;
  top: -20px;
  right: -20px;
}

.spool-card p {
  margin: 8px 0;
}

.pagination-controls {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 16px 0;
}

.pagination-controls span {
  margin: 0 16px;
}
</style>
