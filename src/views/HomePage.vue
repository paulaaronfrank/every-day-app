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
        <ion-item v-for="(item, index) in items" :key="index">
          <!-- Apply 'crossed-out' class conditionally -->
          <ion-label :class="{ 'crossed-out': item.checked }">{{
            item.name
          }}</ion-label>
          <ion-checkbox
            slot="start"
            :value="item.checked"
            @ionChange="toggleItem(index)"
          />
        </ion-item>
      </ion-list>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { ref } from "vue";
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
  name: string;
  checked: boolean;
}

const items = ref<Item[]>([]);

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
        handler: (data) => {
          if (data.name) {
            // Ensure the name is not empty
            items.value.push({ name: data.name, checked: false });
          }
        },
      },
    ],
  });

  await alert.present();
};

// Updated toggleItem to use Vue's reactivity system more effectively
const toggleItem = (index: number) => {
  items.value[index] = {
    ...items.value[index],
    checked: !items.value[index].checked,
  };
};
</script>

<style scoped>
.crossed-out {
  text-decoration: line-through;
}
</style>
