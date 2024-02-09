<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Every Day</ion-title>
        <ion-buttons slot="end">
          <ion-button @click="showAddItemPrompt()">
            <ion-icon slot="icon-only" :icon="addOutline"></ion-icon>
          </ion-button>
        </ion-buttons>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-list>
        <ion-item
          v-for="item in items.filter((item) => !item.deleted)"
          :key="item.id"
          @click="toggleCheckOff(item.id)"
        >
          <ion-label>
            <div :class="{ 'checked-today': isCheckedToday(item.checkedDays) }">
              {{ item.name }}
            </div>
            <div style="font-size: small; color: grey">
              Total: {{ calculateTotalDays(item.checkedDays) }}, Streak:
              {{ calculateStreak(item.checkedDays) }}
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
import { onMounted, ref } from "vue";
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
}

const items = ref<Item[]>([]);

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

const addItem = async (name: string) => {
  const newItem: Item = {
    id: Date.now().toString(),
    name,
    deleted: false,
    checkedDays: [], // Initialize with an empty array
  };
  items.value.push(newItem);
  await saveItems();
};

onMounted(() => {
  loadItems();
});

const showAddItemPrompt = async () => {
  const alert = await alertController.create({
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
        text: "Add",
        handler: async (data) => {
          if (data.name) {
            // Ensure the name is not empty
            await addItem(data.name);
          }
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

const calculateStreak = (checkedDays: number[]) => {
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
</script>

<style scoped>
.checked-today {
  text-decoration: line-through;
}
</style>
