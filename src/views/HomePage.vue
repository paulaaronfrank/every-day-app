<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>{{ headerTitle }}</ion-title>
        <ion-buttons slot="end">
          <ion-button @click="showAddItemPrompt()">
            <ion-icon slot="icon-only" :icon="addOutline"></ion-icon>
          </ion-button>
        </ion-buttons>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true" ref="content">
      <ion-list>
        <ion-item
          v-for="item in dailyItems"
          :key="item.id"
          @click="toggleCheckOff(item.id)"
        >
          <ion-label>
            <div :class="{ 'checked-today': isCheckedToday(item.checkedDays) }">
              {{ item.name }}
            </div>
            <div style="font-size: small; color: grey">
              Total: {{ calculateTotalDays(item.checkedDays) }}, Streak:
              {{ calculateDailyStreak(item.checkedDays) }}, Rate:
              {{ calculateDailyPercentile(item.checkedDays) }}%
            </div>
          </ion-label>
          <ion-checkbox
            slot="start"
            :checked="isCheckedToday(item.checkedDays)"
          />
        </ion-item>
      </ion-list>
      <div
        v-if="weeklyItems.length > 0"
        class="weekly-goals-divider"
        ref="divider"
      >
        <ion-toolbar>
          <ion-title>Every Week</ion-title>
        </ion-toolbar>
      </div>
      <ion-list>
        <ion-item
          v-for="item in weeklyItems"
          :key="item.id"
          @click="toggleCheckOff(item.id)"
        >
          <ion-label>
            <div :class="{ 'checked-today': isCheckedToday(item.checkedDays) }">
              {{ item.name }}
            </div>
            <div style="font-size: small; color: grey">
              Total: {{ calculateTotalDays(item.checkedDays) }}, Streak:
              {{ calculateWeeklyStreak(item.checkedDays) }}, Rate:
              {{ calculateWeeklyPercentile(item.checkedDays) }}%
            </div>
          </ion-label>
          <ion-checkbox
            slot="start"
            :checked="isCheckedToday(item.checkedDays)"
          />
        </ion-item>
      </ion-list>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { onMounted, ref, computed } from "vue";
import { Preferences } from "@capacitor/preferences";
import {
  IonContent,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
  IonList,
  IonItem,
  IonLabel,
  IonCheckbox,
  IonButtons,
  IonButton,
  IonIcon,
  alertController,
} from "@ionic/vue";
import { addOutline } from "ionicons/icons";

interface Item {
  id: string;
  name: string;
  deleted: boolean;
  checkedDays: number[]; // Array of Unix timestamps
  everyWeek: boolean;
}

const headerTitle = ref("Every Day");

const items = ref<Item[]>([]);

const dailyItems = computed(() => {
  return items.value.filter((item) => !item.deleted && !item.everyWeek);
});

const weeklyItems = computed(() => {
  return items.value.filter((item) => !item.deleted && item.everyWeek);
});

const loadItems = async () => {
  const { value } = await Preferences.get({ key: "items" });
  items.value = value ? JSON.parse(value) : [];
};

const saveItems = async () => {
  await Preferences.set({
    key: "items",
    value: JSON.stringify(items.value.filter((item) => !item.deleted)), // Optionally filter out deleted items before saving
  });
};

const addItem = async (name: string, everyWeek: boolean) => {
  const newItem: Item = {
    id: Date.now().toString(),
    name,
    deleted: false,
    checkedDays: [],
    everyWeek, // Set the everyWeek property based on the parameter
  };
  items.value.push(newItem);
  await saveItems();
};

onMounted(() => {
  loadItems();
});

const showAddItemPrompt = async () => {
  // First, ask for the item name
  let alert = await alertController.create({
    header: "Add New Item",
    inputs: [
      {
        name: "name",
        type: "text",
        placeholder: "Item name",
      },
    ],
    buttons: [
      {
        text: "Cancel",
        role: "cancel",
      },
      {
        text: "Next",
        handler: async (data) => {
          if (data.name) {
            // Next, confirm if it should repeat every week
            showRepeatWeeklyPrompt(data.name);
          }
        },
      },
    ],
  });

  await alert.present();
};

const showRepeatWeeklyPrompt = async (name: string) => {
  let alert = await alertController.create({
    header: "Every Day?",
    message: "We love ambition! Do you want daily or weekly repition stats?",
    buttons: [
      {
        text: "Daily",
        role: "cancel",
        handler: () => {
          addItem(name, false);
        },
      },
      {
        text: "Weekly",
        handler: () => {
          addItem(name, true);
        },
      },
    ],
  });

  await alert.present();
};

const toggleCheckOff = async (id: string) => {
  const item = items.value.find((item) => item.id === id);
  if (!item) return;

  const now = new Date(); // Current timestamp without normalization
  const today = new Date(now.getFullYear(), now.getMonth(), now.getDate()); // Normalized to start of the day for comparison
  const todayTimestamp = today.getTime();

  const lastCheckIndex = item.checkedDays.findIndex((timestamp) => {
    const checkDate = new Date(timestamp);
    const startOfCheckDate = new Date(
      checkDate.getFullYear(),
      checkDate.getMonth(),
      checkDate.getDate()
    );
    return startOfCheckDate.getTime() === todayTimestamp;
  });

  if (lastCheckIndex !== -1) {
    // Uncheck: Remove today's timestamp if it's the last in the list
    item.checkedDays.splice(lastCheckIndex, 1);
  } else {
    // Check: Add current timestamp without normalization
    item.checkedDays.push(now.getTime());
  }

  await saveItems();
};

const isCheckedToday = (checkedDays: number[]) => {
  const today = new Date();
  const startOfToday = new Date(
    today.getFullYear(),
    today.getMonth(),
    today.getDate()
  ).getTime();

  return checkedDays.some((timestamp) => {
    const checkDate = new Date(timestamp);
    const startOfCheckDate = new Date(
      checkDate.getFullYear(),
      checkDate.getMonth(),
      checkDate.getDate()
    ).getTime();
    return startOfCheckDate === startOfToday;
  });
};

const calculateTotalDays = (checkedDays: number[]) => {
  return checkedDays.length;
};

const calculateDailyStreak = (checkedDays: number[]) => {
  let streak = 0;
  let today = new Date();
  today = new Date(today.getFullYear(), today.getMonth(), today.getDate()); // Normalize to start of the day

  let checkingDay = today;
  let foundToday = false;

  // Loop backwards through the sorted list of checked days
  const sortedCheckedDays = checkedDays.slice().sort((a, b) => b - a); // Sort in descending order

  for (const timestamp of sortedCheckedDays) {
    const checkDate = new Date(timestamp);
    const normalizedCheckDate = new Date(
      checkDate.getFullYear(),
      checkDate.getMonth(),
      checkDate.getDate()
    );

    if (normalizedCheckDate.getTime() === checkingDay.getTime()) {
      if (!foundToday) {
        foundToday = true; // Today is included in the streak
      }
      streak++;
      // Move to the previous day
      checkingDay.setDate(checkingDay.getDate() - 1);
    } else if (
      foundToday ||
      normalizedCheckDate.getTime() < checkingDay.getTime()
    ) {
      // If we already found today in the list or the check date is before the checking day, break the loop
      break;
    }
  }

  return streak;
};

const calculateWeeklyStreak = (checkedDays: number[]) => {
  if (checkedDays.length === 0) return 0;

  const weeks = checkedDays.map((timestamp) => {
    const date = new Date(timestamp);
    const startOfYear = new Date(date.getFullYear(), 0, 0);
    const diff = date - startOfYear;
    const oneDay = 1000 * 60 * 60 * 24;
    const dayOfYear = Math.floor(diff / oneDay);
    return Math.floor(dayOfYear / 7);
  });

  const uniqueWeeks = [...new Set(weeks)].sort((a, b) => a - b);
  let streak = 1; // Assuming having any week means a streak of at least 1
  for (let i = 1; i < uniqueWeeks.length; i++) {
    if (uniqueWeeks[i] - uniqueWeeks[i - 1] === 1) {
      streak++;
    } else {
      streak = 1; // Reset streak if there's a gap
    }
  }

  return streak;
};

const calculateDailyPercentile = (checkedDays: number[]) => {
  if (checkedDays.length === 0) return 0;

  const firstCheckIn = new Date(Math.min(...checkedDays));
  const today = new Date();
  const totalDays = Math.ceil((today - firstCheckIn) / (1000 * 60 * 60 * 24));
  const activeDays = checkedDays.length;

  return (activeDays / totalDays) * 100;
};

const calculateWeeklyPercentile = (checkedDays: number[]) => {
  if (checkedDays.length === 0) return 0;

  const weeksMap = new Map();
  checkedDays.forEach((timestamp) => {
    const date = new Date(timestamp);
    const weekYear = getWeekNumber(date);
    weeksMap.set(weekYear, (weeksMap.get(weekYear) || 0) + 1);
  });

  const firstCheckIn = new Date(Math.min(...checkedDays));
  const today = new Date();
  const totalWeeks = Math.ceil(
    (today - firstCheckIn) / (1000 * 60 * 60 * 24) / 7
  );
  const totalCheckIns = Array.from(weeksMap.values()).reduce(
    (acc, val) => acc + val,
    0
  );

  return (totalCheckIns / totalWeeks) * 100;
};

// Helper function to get the ISO week number
function getWeekNumber(d: Date) {
  d = new Date(Date.UTC(d.getFullYear(), d.getMonth(), d.getDate()));
  d.setUTCDate(d.getUTCDate() + 4 - (d.getUTCDay() || 7));
  const yearStart = new Date(Date.UTC(d.getUTCFullYear(), 0, 1));
  const weekNo = Math.ceil(((d - yearStart) / 86400000 + 1) / 7);
  return d.getUTCFullYear() + "-W" + weekNo;
}
</script>

<style scoped>
.checked-today {
  text-decoration: line-through;
}
</style>
