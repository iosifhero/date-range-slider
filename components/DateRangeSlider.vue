<template>
  <div class="date-slider">
    <div class="view-toggle">
      <button
        :class="{ active: viewMode === 'years' }"
        @click="switchView('years')"
      >
        Все года
      </button>
      <button
        :class="{ active: viewMode === 'months' }"
        @click="switchView('months')"
      >
        Месяца
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
          <div class="tooltip tooltip_right">
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
            'hide-on-mobile': !tick.isYear && isMobile,
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

const props = defineProps({
  minDate: {
    type: Date,
    required: true,
  },
  maxDate: {
    type: Date,
    required: true,
  },
  startDate: {
    type: Date,
    default: null,
  },
  endDate: {
    type: Date,
    default: null,
  },
});

const slider = ref(null);
const viewMode = ref("years");
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

const isMobile = ref(window.innerWidth <= 768);

onMounted(() => {
  window.addEventListener("mousemove", onMouseMove);
  window.addEventListener("mouseup", onMouseUp);
  window.addEventListener("resize", () => {
    isMobile.value = window.innerWidth <= 768;
  });
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

  if (dragging.value === "start" && value < endValue.value) {
    updateDate(startDate, value);
  } else if (dragging.value === "end" && value > startValue.value) {
    updateDate(endDate, value);
  }
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

const formatDate = (date) => {
  const month = monthNames[date.getMonth()];
  const year = date.getFullYear();
  return `${month} ${year}`;
};

const generateTicks = () => {
  const ticksArray = [];
  const step = viewMode.value === "years" ? 12 : 1;
  const start = minDateValue.value;
  const end = maxDateValue.value;

  for (let i = start; i <= end; i += step) {
    const position = ((i - start) / totalRange.value) * 100;
    const isYear = i % 12 === 0;
    const label = formatTickLabel(i);

    ticksArray.push({ position, label, isYear });
  }

  return ticksArray;
};

const formatTickLabel = (monthValue) => {
  const year = Math.floor(monthValue / 12);
  const monthIndex = monthValue % 12;

  if (viewMode.value === "years" || monthIndex === 0) {
    return year;
  } else {
    return monthNames[monthIndex].substring(0, 3);
  }
};
</script>

<style scoped>
.date-slider {
  user-select: none;
  display: flex;
  justify-content: space-between;
  padding: 50px 20px;
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
  width: 11px;
  height: 11px;
  background-color: #fff;
  border: 5px solid #6ea6f7;
  border-radius: 50%;
  cursor: pointer;
  top: 50%;
  transform: translate(-50%, -50%);
  z-index: 1;
}
.tooltip {
  position: absolute;
  top: -67px;
  left: 50%;
  transform: translateX(-50%);
  background-color: #fff;
  color: #6ea6f7;
  border-radius: 4px;
  padding: 6px 14px;
  text-align: center;
  box-shadow: 0px 0px 6px rgba(0, 0, 0, 0.1);
  word-wrap: break-word;
  max-width: 100px;
  z-index: 2;
}
.tooltip_right {
  top: 20px;
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
.hide-on-mobile {
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
    top: -55px;
    font-size: 12px;
    max-width: 100px;
  }
  .tooltip_right {
    top: 20px;
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
    top: -50px;
    font-size: 10px;
    max-width: 80px;
  }
  .tooltip_right {
    top: 20px;
  }

  .ticks {
    font-size: 10px;
  }
}
</style>
