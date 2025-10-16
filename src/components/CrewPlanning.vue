<template>
  <div class="min-vh-100 bg-light d-flex flex-column">
    <!-- Header -->
    <div class="bg-dark text-white p-3 shadow d-flex align-items-center">
      <h1 class="h5 fw-semibold m-0">Crew Planning Dashboard</h1>
      <div class="ms-auto d-flex align-items-center gap-2">
        <input type="month" v-model="startMonthStr" class="form-control form-control-sm" style="width: 160px" />
        <select v-model.number="monthCount" class="form-select form-select-sm" style="width: 100px">
          <option :value="6">6</option>
          <option :value="9">9</option>
          <option :value="12">12</option>
          <option :value="15">15</option>
        </select>
      </div>
    </div>

    <div class="d-flex overflow-hidden">
      <div class="bg-secondary border-end overflow-auto" style="width: 16rem">
        <div class="position-sticky top-0 p-3" style="z-index: 20">
          <div class="d-flex justify-content-between small fw-semibold text-secondary-emphasis">
            <span>Vessel</span>
            <span>Rank</span>
          </div>
        </div>
      </div>
      <div class="flex-grow-1 d-flex flex-column overflow-hidden position-relative">
        <div class="bg-white border-bottom position-sticky top-0" style="z-index: 10">
          <div class="d-flex" :style="{ marginLeft: -scrollLeft + 'px' }">
            <div v-for="(m, idx) in monthsBar" :key="idx"
              class="border-end text-center bg-light flex-shrink-0 d-flex flex-column justify-content-center"
              :style="{ width: Math.max(m.days * dayWidth, 100) + 'px', height: '50px' }">
              <div class="small fw-semibold text-secondary">
                {{ m.monthName }}
              </div>
              <div v-if="spanMultipleYears" class="small text-secondary">
                {{ m.year }}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Schedule alanı -->
    <div class="d-flex overflow-auto" style="border-top: 1px solid #dee2e6; height: 500px; width: 100%">
      <!-- Left: Vessels / Ranks -->
      <!-- Vessel Alanı -->
      <div ref="vesselRef" class="bg-secondary-subtle border-end overflow-hidden" style="width: 16rem"
        @scroll="onVesselScroll">
        <div v-for="(row, idx) in allRows" :key="idx" class="border-bottom" :style="{ height: row.height + 'px' }">
          <div v-if="row.type === 'vessel'"
            class="d-flex align-items-center justify-content-between p-3 bg-light cursor-pointer"
            @click="toggleVessel(row.vessel.name)">
            <div class="d-flex align-items-center gap-2">
              <font-awesome-icon v-if="expandedVessels[row.vessel.name]" icon="chevron-down" class="text-secondary" />
              <font-awesome-icon v-else icon="chevron-right" class="text-secondary" />
              <span class="fw-medium small text-dark" :title="row.vessel.name"
                style="text-overflow: ellipsis; white-space: nowrap; overflow: hidden; display: inline-block; max-width: 100%;">{{
                  row.vessel.name }}</span>
            </div>
          </div>
          <div v-if="row.type === 'rank'" class="d-flex align-items-center p-3 ps-4 border-top bg-white">
            <font-awesome-icon icon="user" class="text-secondary me-2" />
            <span class="small text-secondary" :title="row.rank.title"
              style="text-overflow: ellipsis; white-space: nowrap; overflow: hidden; display: inline-block; max-width: 100%;">{{
                row.rank.title }}</span>
          </div>
        </div>
      </div>

      <!-- TimeLine Alanı -->
      <div class="flex-grow-1 d-flex flex-column overflow-hidden position-relative">
        <div ref="timelineRef" class="flex-grow-1 overflow-auto" @scroll="onScroll">
          <div :style="{ width: totalDays * dayWidth + 'px', minWidth: '100%' }">
            <div v-for="(row, idx) in allRows" :key="idx" class="border-bottom" :style="{ height: row.height + 'px' }">
              <div v-if="row.type === 'rank'" class="position-relative border-top bg-white">
                <div v-for="(assignment, aIdx) in row.rank.assignments" :key="aIdx"
                  class="position-absolute text-white small px-2 py-1 rounded shadow-sm d-flex align-items-center justify-content-between overflow-hidden cursor-pointer"
                  :class="getStatusColor(assignment.status)"
                  :style="{ left: assignment.start * dayWidth + 'px', width: (assignment.renderDuration || assignment.duration) * dayWidth + 'px', top: assignment.row * rowHeight + 4 + 'px', height: '40px' }"
                  @mouseenter="(e) => handleMouseEnter(e, assignment)" @mouseleave="handleMouseLeave">
                  <span class="text-truncate fw-medium">{{ assignment.name || "Unassigned" }}</span>
                  <span class="ms-2 small opacity-75 text-nowrap">{{ assignment.renderDuration || assignment.duration
                  }}d</span>
                  <div class="position-absolute end-0 top-0 bottom-0" style="width: 8px; cursor: ew-resize"
                    @mousedown="(e) => handleResizeStart(e, row.vIdx, row.rIdx, aIdx, assignment)" />
                  <div v-if="assignment._ongoing" class="position-absolute"
                    style="right: 10px; top: 50%; transform: translateY(-50%)">
                    <font-awesome-icon icon="arrow-right" />
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div v-if="tooltip" class="position-fixed bg-white shadow rounded p-3 border" :style="{
          left: tooltip.x + 'px',
          top: tooltip.y + 'px',
          transform: 'translate(-50%, -100%)',
          minWidth: '280px',
          zIndex: 1050,
          pointerEvents: 'none',
        }">
          <div class="d-flex gap-3">
            <div class="flex-shrink-0">
              <div class="rounded-circle d-flex align-items-center justify-content-center text-white fw-semibold" style="
                  width: 48px;
                  height: 48px;
                  background: linear-gradient(135deg, #60a5fa, #2563eb);
                ">
                {{
                  tooltip.data.name
                    ? tooltip.data.name.substring(0, 2).toUpperCase()
                    : "UN"
                }}
              </div>
            </div>

            <div class="flex-grow-1">
              <div class="fw-semibold text-dark mb-2">
                {{ tooltip.data.name || "Unassigned" }}
              </div>
              <div class="small text-secondary">
                <div class="d-flex justify-content-between">
                  <span class="fw-medium">Start Date:</span>
                  <span>{{ formatDate(tooltip.data.startDate) }}</span>
                </div>
                <div class="d-flex justify-content-between">
                  <span class="fw-medium">End Date:</span>
                  <span>{{
                    formatDate(
                      calculateEndDate(
                        tooltip.data.startDate,
                        tooltip.data.duration
                      )
                    )
                  }}</span>
                </div>
                <div class="d-flex justify-content-between">
                  <span class="fw-medium">Duration:</span>
                  <span>{{ tooltip.data.duration }} days</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  name: "CrewPlanning",
  data() {
    return {
      expandedVessels: {},
      scrollLeft: 0,
      tooltip: null,
      resizing: null,

      dayWidth: 3,
      rowHeight: 48,
      startMonthStr: "",
      monthCount: 12,

      vessels: [],
    };
  },
  computed: {
    processedVessels() {
      const deep = JSON.parse(JSON.stringify(this.vessels));
      const timelineStart = this.timelineStartDate;
      const timelineEnd = this.timelineEndDate;
      deep.forEach((v) => {
        v.ranks.forEach((r) => {
          const result = [];
          r.assignments.forEach((a, origIdx) => {
            if (new Date(a.startDate) > timelineEnd) return;
            const msDay = 1000 * 60 * 60 * 24;
            const startDate = new Date(a.startDate);
            const endDate = new Date(
              a.endDate || this.calculateEndDate(a.startDate, a.duration)
            );

            const effectiveStart = new Date(Math.max(startDate, timelineStart));
            const effectiveEnd = new Date(Math.min(endDate, timelineEnd));

            const startOffsetDays = Math.max(
              0,
              Math.floor((effectiveStart - timelineStart) / msDay)
            );
            const renderDays = Math.max(
              1,
              Math.floor((effectiveEnd - effectiveStart) / msDay) + 1
            );

            a.start = startOffsetDays;
            a.renderDuration = renderDays;
            a._ongoing = endDate > timelineEnd;
            a._origIdx = origIdx;
            result.push(a);
          });
          // overlap satırları, filtrelenmiş dizi üzerinde hesapla
          result.forEach((a, idx) => {
            a.row = this.calculateRowForAssignment(result, idx);
          });
        });
      });
      return deep;
    },
    allRows() {
      const rows = [];
      let vIdx = 0;
      this.processedVessels.forEach(v => {
        rows.push({ type: 'vessel', vessel: v, height: 48, vIdx });
        if (this.expandedVessels[v.name]) {
          let rIdx = 0;
          v.ranks.forEach(r => {
            rows.push({ type: 'rank', vessel: v, rank: r, height: this.getMaxRowsForRank(r.assignments) * this.rowHeight, vIdx, rIdx });
            rIdx++;
          });
        }
        vIdx++;
      });
      return rows;
    },
    monthsBar() {
      if (!this.startMonthStr) return [];
      const [y, m] = this.startMonthStr.split("-").map((x) => parseInt(x, 10));
      const items = [];
      for (let i = 0; i < this.monthCount; i++) {
        const date = new Date(y, m - 1 + i, 1);
        const year = date.getFullYear();
        const month = date.getMonth();
        const days = new Date(year, month + 1, 0).getDate();
        const monthName = date.toLocaleString("en-US", { month: "long" });
        items.push({ year, month, days, monthName });
      }
      return items;
    },
    spanMultipleYears() {
      if (this.monthsBar.length === 0) return false;
      const firstYear = this.monthsBar[0].year;
      const lastYear = this.monthsBar[this.monthsBar.length - 1].year;
      return firstYear !== lastYear;
    },
    totalDays() {
      return this.monthsBar.reduce((sum, m) => sum + m.days, 0);
    },
    timelineStartDate() {
      if (this.monthsBar.length === 0) return null;
      const first = this.monthsBar[0];
      return new Date(first.year, first.month, 1);
    },
    timelineEndDate() {
      if (this.monthsBar.length === 0) return null;
      const last = this.monthsBar[this.monthsBar.length - 1];
      return new Date(last.year, last.month, last.days);
    },
  },
  methods: {
    async loadData() {
      const res = await fetch("/data.json");
      const items = await res.json();
      const grouped = {};
      items.forEach((it) => {
        const vesselName = it.VesselName || it.vesselName || "Unknown Vessel";
        const rankTitle =
          it.CrewWorkPositionName || it.crewWorkPositionName || "Unknown Rank";
        if (!grouped[vesselName]) grouped[vesselName] = {};
        if (!grouped[vesselName][rankTitle])
          grouped[vesselName][rankTitle] = [];

        const msDay = 1000 * 60 * 60 * 24;
        const startStr = it.StartDate || it.startDate;
        const endStr = it.EndDate || it.endDate;
        const start = new Date(startStr);
        const end = new Date(endStr);
        const duration = Math.max(1, Math.floor((end - start) / msDay) + 1);

        const typeVal = it.Type !== undefined ? it.Type : it.type;
        let status = "active";
        if (typeof typeVal === "number") {
          status =
            typeVal === 1 ? "active" : typeVal === 2 ? "future" : "active";
        } else if (typeof typeVal === "string") {
          status = typeVal.toLowerCase();
        }

        grouped[vesselName][rankTitle].push({
          name: it.Subject || it.subject || "",
          start: 0,
          duration,
          status,
          startDate: startStr,
          endDate: endStr,
        });
      });

      const vessels = Object.keys(grouped).map((vesselName) => ({
        name: vesselName,
        ranks: Object.keys(grouped[vesselName]).map((rankTitle) => ({
          title: rankTitle,
          assignments: grouped[vesselName][rankTitle],
        })),
      }));
      this.vessels = vessels;
    },
    toggleVessel(vesselName) {
      this.$set(
        this.expandedVessels,
        vesselName,
        !this.expandedVessels[vesselName]
      );
    },
    onScroll() {
      const el = this.$refs.timelineRef;
      if (el) {
        this.scrollLeft = el.scrollLeft;
        if (this.$refs.vesselRef) {
          this.$refs.vesselRef.scrollTop = el.scrollTop;
        }
      }
    },
    onVesselScroll() {
      const el = this.$refs.vesselRef;
      if (el && this.$refs.timelineRef) {
        this.$refs.timelineRef.scrollTop = el.scrollTop;
      }
    },
    getStatusColor(status) {
      switch (status) {
        case "active":
          return "bg-dark";
        case "future":
          return "bg-primary";
        case "error":
          return "bg-danger-subtle text-dark";
        case "inactive":
          return "bg-secondary";
        default:
          return "bg-dark";
      }
    },
    getMaxRowsForRank(assignments) {
      if (!assignments || assignments.length === 0) return 1;
      const maxRow = Math.max(...assignments.map((a) => a.row || 0));
      return maxRow + 1;
    },
    calculateRowForAssignment(assignments, currentIndex) {
      const current = assignments[currentIndex];
      const currentStart = current.start;
      const currentDuration = current.renderDuration || current.duration;
      const currentEnd = current.start + currentDuration;

      let row = 0;
      const usedRows = new Set();

      for (let i = 0; i < currentIndex; i++) {
        const other = assignments[i];
        const otherStart = other.start;
        const otherDuration = other.renderDuration || other.duration;
        const otherEnd = other.start + otherDuration;
        const hasOverlap = !(
          currentEnd <= otherStart || currentStart >= otherEnd
        );
        if (hasOverlap) {
          usedRows.add(other.row || 0);
        }
      }

      while (usedRows.has(row)) row++;
      return row;
    },
    calculateEndDate(startDate, duration) {
      const date = new Date(startDate);
      date.setDate(date.getDate() + duration);
      return date.toISOString().split("T")[0];
    },
    formatDate(dateStr) {
      const date = new Date(dateStr);
      return date.toLocaleDateString("en-US", {
        month: "short",
        day: "numeric",
        year: "numeric",
      });
    },
    handleMouseEnter(e, assignment) {
      const rect = e.currentTarget.getBoundingClientRect();
      this.tooltip = {
        x: rect.left + rect.width / 2,
        y: rect.top - 10,
        data: assignment,
      };
    },
    handleMouseLeave() {
      if (!this.resizing) {
        this.tooltip = null;
      }
    },
    handleResizeStart(e, vesselIdx, rankIdx, assignmentIdx, assignment) {
      e.stopPropagation();
      this.resizing = {
        vesselIdx,
        rankIdx,
        assignmentIdx,
        origIdx: assignment._origIdx,
        initialWidth: assignment.duration,
        initialX: e.clientX,
      };
      // global mouse move / up
      document.addEventListener("mousemove", this._onMouseMove);
      document.addEventListener("mouseup", this._onMouseUp);
    },
    _onMouseMove(e) {
      if (!this.resizing) return;
      const {
        vesselIdx,
        rankIdx,
        assignmentIdx,
        origIdx,
        initialWidth,
        initialX,
      } = this.resizing;
      const deltaX = e.clientX - initialX;
      const deltaDays = Math.round(deltaX / this.dayWidth);
      const newDuration = Math.max(7, initialWidth + deltaDays);

      // immutability gerekmeden doğrudan değiştiriyoruz (Vue 2 reactivity için set kullan)
      const v = this.vessels[vesselIdx];
      const a =
        v.ranks[rankIdx].assignments[origIdx != null ? origIdx : assignmentIdx];
      this.$set(a, "duration", newDuration);
    },
    _onMouseUp() {
      this.resizing = null;
      document.removeEventListener("mousemove", this._onMouseMove);
      document.removeEventListener("mouseup", this._onMouseUp);
    },
  },
  mounted() {
    this._onMouseMove = this._onMouseMove.bind(this);
    this._onMouseUp = this._onMouseUp.bind(this);
    const now = new Date();
    const m = new Date(now.getFullYear(), now.getMonth() - 3, 1);
    const yyyy = m.getFullYear();
    const mm = String(m.getMonth() + 1).padStart(2, "0");
    this.startMonthStr = `${yyyy}-${mm}`;
    this.loadData();
  },
  beforeDestroy() {
    document.removeEventListener("mousemove", this._onMouseMove);
    document.removeEventListener("mouseup", this._onMouseUp);
  },
};
</script>
