<script>
export default {
  name: 'MyDay',
};
</script>

<script setup>
import {ref, onActivated, onDeactivated, watchEffect, computed} from 'vue';
import {ElMessage} from 'element-plus';
import {cloneDeep} from 'lodash';
import {Edit, Plus, Check} from '@element-plus/icons';
import {userMyDayKey} from '/@/utils/constants';
import {isObject, formatTimeValue, getFormattedTime} from '/@/utils';
import AddPeriodDialog from './AddPeriod.vue';
import PeriodCard from './PeriodCard.vue';
import NavTopBar from '/@/components/NavTopBar.vue';

import {useStore} from 'vuex';

const store = useStore();

const isPhone = computed(() => store.state.application.isPhone);

const Hour = 60;
const time = ref(0);
const timeFormatted = ref('');

const myDayList = ref([]);

const defaultOneDay = {
  timePoints: [0, 30, Hour, 2 * Hour, 15 * Hour, 16 * Hour],
  eventsName: [null, '起床', null, '学习', null, '洗漱'],
  wakeAndSleep: {
    startTime: 8 * Hour,
    startLabel: '起床啦',
    endTime: 16 * Hour,
    endLabel: '睡觉啦',
  },
};

const myDayConfigLocal = JSON.parse(localStorage.getItem(userMyDayKey));


if (isObject(myDayConfigLocal)) {
  // old version 0.0.1
  const wakeAndSleep = {
    startTime: myDayConfigLocal.startTime,
    startLabel: myDayConfigLocal.startLabel,
    endTime: myDayConfigLocal.endTime,
    endLabel: myDayConfigLocal.endLabel,
  };
  myDayList.value = [{
    timePoints: myDayConfigLocal.timePoints,
    eventsName: myDayConfigLocal.eventsName,
    wakeAndSleep,
  }];
} else {
  // new version 0.1.0
  if (Array.isArray(myDayConfigLocal)) {
    myDayList.value = myDayConfigLocal;
  } else {
    // default(init) value
    myDayList.value = [cloneDeep(defaultOneDay)];
  }
}
// insert key
myDayList.value.forEach(day => {
  day.key_id = Symbol();
});

watchEffect(() => {
  localStorage.setItem(userMyDayKey, JSON.stringify(myDayList.value));
});

// get now time
let refreshTimeInterval = 0;
const getNowTime = () => {
  const now = new Date();
  time.value = now.getHours() * Hour + now.getMinutes() + now.getSeconds() / 60;
  timeFormatted.value = getFormattedTime();
};
onActivated(() => {
  getNowTime();
  refreshTimeInterval = setInterval(getNowTime, 1000);
});
onDeactivated(() => clearInterval(refreshTimeInterval));

/**
 * margeEventPoints
 */
const mergeNull = (dayIndex) => {
  const timePoints = myDayList.value[dayIndex].timePoints;
  const eventsName = myDayList.value[dayIndex].eventsName;
  const wakeAndSleep = myDayList.value[dayIndex].wakeAndSleep;
  if (timePoints[timePoints.length - 1] < wakeAndSleep.endTime) {
    timePoints.push(wakeAndSleep.endTime);
    eventsName.push(null);
  }
  for (let i = 1; i < timePoints.length; ++i) {
    if (eventsName[i - 1] === eventsName[i] && i > 1) {
      timePoints.splice(i - 1, 1);
      eventsName.splice(i - 1, 1);
      i = i - 2;
      continue;
    }
    if (timePoints[i - 1] >= timePoints[i]) {
      timePoints.splice(i - 1, 1);
      eventsName.splice(eventsName[i - 1] === null ? i - 1 : i, 1);
      i = i - 2;
      continue;
    }
    if (timePoints[i] > wakeAndSleep.endTime) {
      timePoints.splice(i, 1);
      eventsName.splice(i, 1);
      --i;
    }
  }
};

// add period
const addPeriodDialogRef = ref(null);
const addPeriodDialogDayIndex = ref(0);
const addPeriodDialogPeriodIndex = ref(0);

// open dialog
const openAddDialog = (dayIndex, periodIndex) => {
  addPeriodDialogDayIndex.value = dayIndex;
  addPeriodDialogPeriodIndex.value = periodIndex;
  const timePoints = myDayList.value[dayIndex].timePoints;
  const wakeAndSleep = myDayList.value[dayIndex].wakeAndSleep;
  let min = timePoints[periodIndex - 1];
  let max = timePoints[periodIndex];
  const config = {
    startTime: wakeAndSleep.startTime,
    type: 'add',
    min: min,
    max: max,
    timeRange: [min, max],
  };
  addPeriodDialogRef.value.openDialog(config);
};

const openEditDialog = (dayIndex, periodIndex) => {
  addPeriodDialogDayIndex.value = dayIndex;
  addPeriodDialogPeriodIndex.value = periodIndex;
  const wakeAndSleep = myDayList.value[dayIndex].wakeAndSleep;
  const timePoints = myDayList.value[dayIndex].timePoints;
  const eventsName = myDayList.value[dayIndex].eventsName;
  let min = eventsName[periodIndex - 1] === null ? timePoints[periodIndex - 2] : timePoints[periodIndex - 1];
  let max = eventsName[periodIndex + 1] === null ? timePoints[periodIndex + 1] : timePoints[periodIndex];
  const timeRange = [timePoints[periodIndex - 1], timePoints[periodIndex]];
  const name = eventsName[periodIndex];
  const config = {
    startTime: wakeAndSleep.startTime,
    type: 'edit',
    name,
    min,
    max,
    timeRange,
  };
  addPeriodDialogRef.value.openDialog(config);
};

const openEditStartDialog = (dayIndex) => {
  addPeriodDialogDayIndex.value = dayIndex;
  const timePoints = myDayList.value[dayIndex].timePoints;
  const eventsName = myDayList.value[dayIndex].eventsName;
  const wakeAndSleep = myDayList.value[dayIndex].wakeAndSleep;
  let min;
  if (timePoints.length > 2) {
    min = eventsName[timePoints.length - 1] === null ? formatTimeValue(timePoints[timePoints.length - 2] - wakeAndSleep.endTime) : 0;
  } else {
    min = timePoints[1];
  }
  let max = formatTimeValue(eventsName[1] === null ? timePoints[1] : timePoints[0]);
  let time = 0;
  if (max <= min) {
    max += 24 * Hour;
    time += 24 * Hour;
  }
  if (formatTimeValue(max - min) === 0 && timePoints.length > 2) {
    ElMessage({
      showClose: false,
      dangerouslyUseHTMLString: true,
      message: 'It seems that you cannot change the wake up time yet!<br> Notice that the sleep time will change together! ',
      type: 'warning',
      duration: 5000,
    });
    return;
  }
  const config = {
    startTime: wakeAndSleep.startTime,
    type: 'start',
    name: wakeAndSleep.startLabel,
    min,
    max,
    time,
  };
  addPeriodDialogRef.value.openDialog(config);
};

const openEditEndDialog = (dayIndex) => {
  addPeriodDialogDayIndex.value = dayIndex;
  const timePoints = myDayList.value[dayIndex].timePoints;
  const eventsName = myDayList.value[dayIndex].eventsName;
  const wakeAndSleep = myDayList.value[dayIndex].wakeAndSleep;
  const min = eventsName[timePoints.length - 1] === null ?
    timePoints[timePoints.length - 2] :
    timePoints[timePoints.length - 1];
  const max = 24 * Hour;
  const config = {
    startTime: wakeAndSleep.startTime,
    type: 'end',
    name: wakeAndSleep.endLabel,
    min,
    max,
    time: wakeAndSleep.endTime,
  };
  addPeriodDialogRef.value.openDialog(config);
};

// period actions
const addPeriod = (dayIndex, periodIndex, form) => {
  const timePoints = myDayList.value[dayIndex].timePoints;
  const eventsName = myDayList.value[dayIndex].eventsName;
  const {name, timeRange} = form;
  const [start, end] = timeRange;
  const min = timePoints[periodIndex - 1];
  const max = timePoints[periodIndex];
  if (start === min && end !== max) {
    timePoints.splice(periodIndex, 0, end);
    eventsName.splice(periodIndex, 0, name);
  } else if (start !== min && end !== max) {
    timePoints.splice(periodIndex, 0, start, end);
    eventsName.splice(periodIndex, 0, null, name);
  } else if (start !== min && end === max) {
    timePoints.splice(periodIndex, 0, start);
    eventsName.splice(periodIndex, 1, null, name);
  } else {
    eventsName[periodIndex] = name;
  }
};

const deletePeriod = (dayIndex, periodIndex) => {
  const eventsName = myDayList.value[dayIndex].eventsName;
  eventsName[periodIndex] = null;
  mergeNull(dayIndex);
};

const editPeriod = (dayIndex, periodIndex, form) => {
  const timePoints = myDayList.value[dayIndex].timePoints;
  const eventsName = myDayList.value[dayIndex].eventsName;
  const {name, timeRange, max} = form;
  const [start, end] = timeRange;
  timePoints.splice(periodIndex, 1, start, end, max);
  eventsName.splice(periodIndex, 1, null, name, null);
  mergeNull(dayIndex);
};

const editStart = (dayIndex, form) => {
  const timePoints = myDayList.value[dayIndex].timePoints;
  const eventsName = myDayList.value[dayIndex].eventsName;
  const wakeAndSleep = myDayList.value[dayIndex].wakeAndSleep;
  wakeAndSleep.startLabel = form.name;
  wakeAndSleep.startTime = formatTimeValue(form.time + wakeAndSleep.startTime);
  timePoints.forEach((time, index) => {
    (timePoints[index] = formatTimeValue(time - form.time));
  });
  if (timePoints[0] !== 0) {
    timePoints.unshift(0);
    eventsName.unshift(null);
  }
  mergeNull(dayIndex);
  getNowTime();
};

const editEnd = (dayIndex, form) => {
  const timePoints = myDayList.value[dayIndex].timePoints;
  const eventsName = myDayList.value[dayIndex].eventsName;
  const wakeAndSleep = myDayList.value[dayIndex].wakeAndSleep;
  wakeAndSleep.endLabel = form.name;
  wakeAndSleep.endTime = form.time || 24 * Hour;
  timePoints.push(wakeAndSleep.endTime);
  eventsName.push(null);
  mergeNull(dayIndex);
};

// edit days
const editDays = ref(false);

const addDay = (index) => {
  myDayList.value.splice(index, 0, cloneDeep(defaultOneDay));
};

const deleteDay = (index) => {
  myDayList.value.splice(index, 1);
};

const upDay = (index) => {
  if (index < 1) {
    ElMessage.warning('Now it\'s the first one!');
  } else {
    [myDayList.value[index - 1], myDayList.value[index]] = [myDayList.value[index], myDayList.value[index - 1]];
  }
};

const downDay = (index) => {
  if (index > myDayList.value.length - 2) {
    ElMessage.warning('Now it\'s the last one!');
  } else {
    [myDayList.value[index + 1], myDayList.value[index]] = [myDayList.value[index], myDayList.value[index + 1]];
  }
};

// affix
const affixRef = ref(null);

onActivated(() => {
  affixRef.value && affixRef.value.update();
});
</script>

<template>
  <div
    id="view-my-day"
    :class="[isPhone ? 'is-phone' : 'is-pc', {'editing-day': editDays}]"
  >
    <el-scrollbar view-class="view-my-day-view">
      <nav-top-bar v-if="isPhone" />
      <div class="title-line">
        <h1 class="title">
          {{ timeFormatted }}
        </h1>
        <el-affix
          ref="affixRef"
        >
          <el-icon
            class="edit-days-button"
            @click="editDays = !editDays"
          >
            <template v-if="editDays">
              <check />
            </template>
            <template v-else>
              <edit />
            </template>
          </el-icon>
        </el-affix>
      </div>
      <div
        v-for="(oneDay, index) in myDayList"
        :key="oneDay.key_id"
      >
        <div
          v-if="editDays"
          class="day-action-button center"
          @click="addDay(index)"
        >
          <el-icon>
            <plus />
          </el-icon>
        </div>
        <period-card
          :is-phone="isPhone"
          :time-system="time"
          :time-points="oneDay.timePoints"
          :events-name="oneDay.eventsName"
          :wake-and-sleep="oneDay.wakeAndSleep"
          :other-day="index !== 0 && !editDays"
          :is-editing="editDays"
          @add="openAddDialog(index, $event)"
          @edit="openEditDialog(index, $event)"
          @delete="deletePeriod(index, $event)"
          @start="openEditStartDialog(index)"
          @end="openEditEndDialog(index)"
          @delete-day="deleteDay(index)"
          @up-day="upDay(index)"
          @down-day="downDay(index)"
        />
      </div>
      <div
        v-if="editDays"
        class="day-action-button center"
        @click="addDay(myDayList.length)"
      >
        <el-icon>
          <plus />
        </el-icon>
      </div>
      <add-period-dialog
        ref="addPeriodDialogRef"
        @add="addPeriod(addPeriodDialogDayIndex, addPeriodDialogPeriodIndex, $event)"
        @edit="editPeriod(addPeriodDialogDayIndex, addPeriodDialogPeriodIndex, $event)"
        @start="editStart(addPeriodDialogDayIndex, $event)"
        @end="editEnd(addPeriodDialogDayIndex, $event)"
      />
    </el-scrollbar>
  </div>
</template>

<style lang="scss" scoped>
#view-my-day {
  height: 100%;
  box-sizing: border-box;
  font-size: 18px;

  :deep(.view-my-day-view) {
    max-width: 720px;
    margin: 0 auto;
  }

  .comp-nav-top-bar {
    margin: 0 16px;
    display: flex;
    align-items: center;
    padding: 0 16px;
  }

  &.is-phone {
    .card-block {
      margin-right: 2px;
      margin-left: 2px;
    }

    .period-line {
      justify-content: initial;
    }

    .edit-days-button {
      opacity: initial;
    }
  }

  &.is-pc {
    .edit-days-button:hover {
      opacity: initial;
    }
  }

  &.editing-day {
    .edit-days-button {
      opacity: initial;
    }
  }

  .day-action-button {
    cursor: pointer;
    font-size: 16px;
    margin: 16px;
  }

  .title-line {
    display: flex;
    align-items: center;

    .el-affix {
      width: 40px;
    }
  }

  .title {
    margin-left: 40px;
    width: calc(100% - 40px - 40px);
    text-align: center;
  }

  .edit-days-button {
    padding: 10px;
    transform: translateX(-10px);
    font-size: 20px;
    cursor: pointer;
    opacity: 0.2;
  }
}

:deep(.el-dialog) {
  max-width: 520px;
}

.center {
  display: flex;
  align-items: center;
  justify-content: center;
}
</style>
