<template>
  <div class="date-slider">
    <div v-if="showToggle" class="view-toggle">
      <button
        v-for="option in toggleOptions"
        :key="option.mode"
        :class="{ active: viewMode === option.mode }"
        @click="switchView(option.mode)"
      >
        {{ option.label }}
      </button>
    </div>

    <div class="slider-container">
      <div class="slider" ref="slider">
        <div
          class="track"
          :style="{ width: `${trackWidth}%`, left: `${trackLeft}%` }"
        ></div>
        <div
          class="thumb"
          :style="{ left: `${startThumbPosition}%` }"
          @mousedown="startDrag('start')"
        >
          <div class="tooltip">
            {{ formattedStartDate }}
          </div>
        </div>
        <div
          class="thumb"
          :style="{ left: `${endThumbPosition}%` }"
          @mousedown="startDrag('end')"
        >
          <div class="tooltip">
            <div class="triangle"></div>
            {{ formattedEndDate }}
          </div>
        </div>
      </div>

      <div class="ticks">
        <span
          v-for="tick in ticks"
          :key="tick.position"
          :class="{
            bold: tick.isYear,
            hidden: tick.hidden,
          }"
          :style="{
            left: `${tick.position}%`,
          }"
        >
          {{ tick.label }}
        </span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";

const monthNames = [
  "Январь",
  "Февраль",
  "Март",
  "Апрель",
  "Май",
  "Июнь",
  "Июль",
  "Август",
  "Сентябрь",
  "Октябрь",
  "Ноябрь",
  "Декабрь",
];
const toggleOptions = [
  { mode: "years", label: "Все года" },
  { mode: "months", label: "Месяца" },
];

const props = defineProps({
  minDate: { type: Date, required: true },
  maxDate: { type: Date, required: true },
  startDate: { type: Date, default: null },
  endDate: { type: Date, default: null },
});

const slider = ref(null);
const viewMode = ref("years");
const showToggle = ref(true);
const startDate = ref(props.startDate || props.minDate);
const endDate = ref(props.endDate || props.maxDate);
const dragging = ref(null);

const minDateValue = computed(() =>
  viewMode.value === "years"
    ? props.minDate.getFullYear() * 12
    : getMonthValue(props.minDate)
);
const maxDateValue = computed(() =>
  viewMode.value === "years"
    ? props.maxDate.getFullYear() * 12
    : getMonthValue(props.maxDate)
);
const startValue = computed(() => getMonthValue(startDate.value));
const endValue = computed(() => getMonthValue(endDate.value));
const totalRange = computed(() => maxDateValue.value - minDateValue.value);

const trackLeft = computed(() =>
  Math.max(
    ((startValue.value - minDateValue.value) / totalRange.value) * 100,
    0
  )
);
const trackWidth = computed(() =>
  Math.min(
    ((endValue.value - startValue.value) / totalRange.value) * 100,
    100 - trackLeft.value
  )
);

const startThumbPosition = computed(() => trackLeft.value);
const endThumbPosition = computed(() => trackLeft.value + trackWidth.value);
const formattedStartDate = computed(() => formatDate(startDate.value));
const formattedEndDate = computed(() => formatDate(endDate.value));

const ticks = computed(() => generateTicks());

const minDistanceBetweenTicks = ref(3);

const updateMinDistance = () => {
  const screenWidth = window.innerWidth;
  const rangeYears = (maxDateValue.value - minDateValue.value) / 12;

  if (screenWidth > 1200) {
    minDistanceBetweenTicks.value = rangeYears > 10 ? 5 : 3;
  } else if (screenWidth > 768) {
    minDistanceBetweenTicks.value = rangeYears > 7 ? 6 : 4;
  } else {
    minDistanceBetweenTicks.value = rangeYears > 2 ? 15 : 10;
  }
};

const checkToggleVisibility = () => {
  const totalWidth = slider.value
    ? slider.value.offsetWidth
    : window.innerWidth;
  const tickWidthEstimate = totalWidth / totalRange.value;
  showToggle.value =
    tickWidthEstimate >= minDistanceBetweenTicks.value &&
    totalRange.value <= 120;
};

onMounted(() => {
  window.addEventListener("mousemove", onMouseMove);
  window.addEventListener("mouseup", onMouseUp);
  window.addEventListener("resize", () => {
    updateMinDistance();
    checkToggleVisibility();
  });
  updateMinDistance();
  checkToggleVisibility();
});

const switchView = (mode) => {
  viewMode.value = mode;
};
const startDrag = (type) => {
  dragging.value = type;
};

const onMouseMove = (event) => {
  if (!dragging.value) return;
  const rect = slider.value.getBoundingClientRect();
  const percent = Math.min(
    Math.max(((event.clientX - rect.left) / rect.width) * 100, 0),
    100
  );
  const value =
    minDateValue.value + Math.round((percent / 100) * totalRange.value);

  if (dragging.value === "start" && value < endValue.value)
    updateDate(startDate, value);
  else if (dragging.value === "end" && value > startValue.value)
    updateDate(endDate, value);
};

const onMouseUp = () => {
  dragging.value = null;
};

const updateDate = (dateRef, value) => {
  const year = Math.floor(value / 12);
  const month = value % 12;
  dateRef.value = new Date(year, month, 1);
};

const getMonthValue = (date) => date.getFullYear() * 12 + date.getMonth();
const formatDate = (date) =>
  `${monthNames[date.getMonth()]} ${date.getFullYear()}`;

const generateTicks = () => {
  const ticksArray = [];
  const step = viewMode.value === "years" ? 12 : 1;
  const start = minDateValue.value;
  const end = maxDateValue.value;
  let previousPosition = -Infinity;

  for (let i = start; i <= end; i += step) {
    const position = ((i - start) / totalRange.value) * 100;
    const isYear = i % 12 === 0;
    const label = formatTickLabel(i);

    const hidden =
      position - previousPosition < minDistanceBetweenTicks.value &&
      !(i % 12 === 0);
    if (!hidden || i % 12 === 0) previousPosition = position;

    ticksArray.push({ position, label, isYear, hidden });
  }

  const finalTicksArray = [];
  let lastYearPosition = -Infinity;

  ticksArray.forEach((tick) => {
    if (tick.isYear) {
      if (tick.position - lastYearPosition < minDistanceBetweenTicks.value) {
        tick.hidden = true;
      } else {
        lastYearPosition = tick.position;
      }
    }
    finalTicksArray.push(tick);
  });

  return finalTicksArray;
};

const formatTickLabel = (monthValue) => {
  const year = Math.floor(monthValue / 12);
  const monthIndex = monthValue % 12;
  return viewMode.value === "years" || monthIndex === 0
    ? year
    : monthNames[monthIndex].substring(0, 3);
};
</script>

<style scoped>
.date-slider {
  user-select: none;
  display: flex;
  justify-content: space-between;
  padding: 50px 35px;
  font-family: cursive;
}
.view-toggle button {
  background: none;
  border: none;
  border-bottom: 1px solid currentColor;
  padding: 10px 0 0;
  opacity: 0.5;
  cursor: pointer;
  transition: 0.3s;
  font-weight: bold;
  font-size: 15px;
  color: #2e80f7;
  font-family: cursive;
}
.view-toggle button.active {
  color: #2e80f7;
  border: none;
  opacity: 1;
}
.slider-container {
  width: 100%;
  margin: 20px 0;
}
.slider {
  position: relative;
  height: 6px;
  background-color: #f1eded;
  border-radius: 3px;
}
.track {
  position: absolute;
  height: 100%;
  background-color: #6ea6f7;
}
.thumb {
  position: absolute;
  width: 10px;
  height: 10px;
  background-color: #fff;
  border: 5px solid #6ea6f7;
  border-radius: 50%;
  cursor: pointer;
  top: 50%;
  transform: translate(-50%, -50%);
  z-index: 1;
  display: flex;
  justify-content: center;
}
.tooltip {
  position: absolute;
  top: -71px;
  background-color: #fff;
  color: #6ea6f7;
  border-radius: 10px;
  padding: 6px 14px;
  text-align: center;
  box-shadow: 0px 0px 20px 9px rgba(0, 0, 0, 0.1);
  word-wrap: break-word;
  max-width: 100px;
  z-index: 2;
  display: flex;
  flex-direction: column;
  align-items: center;
}
.tooltip::before {
  content: "";
  position: absolute;
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-top: 8px solid rgb(255, 255, 255);
  top: 98%;
}
.slider .thumb:last-child .tooltip::before {
  border-top: 8px solid transparent;
  border-bottom: 8px solid rgb(255, 255, 255);
  top: -26%;
}
.slider .thumb:last-child .tooltip {
  top: 25px;
}

.ticks {
  position: relative;
  margin-top: 10px;
  font-size: 12px;
}
.ticks span {
  position: absolute;
  transform: translateX(-50%);
  color: #666;
}
.ticks span.bold {
  font-weight: bold;
  font-size: 14px;
}
.ticks span.hidden {
  display: none;
}

@media (max-width: 768px) {
  .date-slider {
    flex-direction: column;
    align-items: center;
  }
  .view-toggle {
    flex-direction: column;
  }

  .view-toggle button {
    padding: 8px;
    font-size: 13px;
  }

  .slider-container {
    padding: 0 10px;
  }

  .thumb {
    width: 12px;
    height: 12px;
  }

  .tooltip {
    top: -60px;
    font-size: 12px;
    max-width: 100px;
  }
  .slider .thumb:last-child .tooltip::before {
    top: -34%;
  }
  .slider .thumb:last-child .tooltip {
    top: 27px;
  }

  .ticks {
    font-size: 12px;
  }
}

@media (max-width: 480px) {
  .view-toggle button {
    padding: 6px;
    font-size: 11px;
  }

  .slider-container {
    padding: 0 5px;
  }

  .thumb {
    width: 10px;
    height: 10px;
  }

  .tooltip {
    top: -55px;
    font-size: 10px;
    max-width: 80px;
  }
  .slider .thumb:last-child .tooltip::before {
    top: -35%;
  }
  .slider .thumb:last-child .tooltip {
    top: 24px;
  }

  .ticks {
    font-size: 10px;
  }
}
</style>
