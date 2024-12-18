<template>
  <v-row no-gutters>
    <v-col class="grow">
      <v-tabs v-model="tabRoad" background-color="transparent" show-arrows>
        <v-tabs-slider />
        <v-tab
          v-for="roadId in Object.keys(roads)"
          :key="roadId"
          :href="`#${roadId}`"
          :data-cy="'roadTab' + roadId"
          @click="store.setActiveRoad(roadId)"
        >
          {{ roads[roadId].name }}
          <v-btn
            v-show="roadId == tabRoad"
            icon
            text
            data-cy="editRoadButton"
            @click="
              newRoadName = roads[roadId].name;
              editDialog = true;
            "
          >
            <v-icon>mdi-pencil</v-icon>
          </v-btn>
        </v-tab>
        <v-dialog
          v-model="editDialog"
          max-width="600"
          data-cy="editRoadDialog"
          @input="newRoadName = ''"
        >
          <v-card>
            <v-btn icon text style="float: right" @click="editDialog = false">
              <v-icon>mdi-close</v-icon>
            </v-btn>
            <v-card-title>Edit Road</v-card-title>
            <v-card-text>
              <v-text-field
                v-if="editDialog"
                v-model="newRoadName"
                autofocus
                label="Road Name"
                data-cy="renameRoadField"
                @keyup.enter="renameRoad"
              />
            </v-card-text>
            <v-card-actions>
              <v-spacer />
              <v-btn
                color="error"
                data-cy="deleteRoadButton"
                @click="
                  editDialog = false;
                  deleteDialog = true;
                "
              >
                <v-icon>mdi-delete</v-icon>
                Delete Road
              </v-btn>
              <v-btn
                color="primary"
                :disabled="otherRoadHasName(tabRoad, newRoadName)"
                data-cy="editRoadSubmitButton"
                @click="renameRoad"
              >
                Submit
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
        <v-dialog
          v-if="tabRoad in roads"
          v-model="deleteDialog"
          max-width="600"
          data-cy="deleteRoadDialog"
        >
          <v-card>
            <v-btn icon text style="float: right" @click="deleteDialog = false">
              <v-icon>mdi-close</v-icon>
            </v-btn>
            <v-card-title
              >Permanently Delete {{ roads[tabRoad].name }}?</v-card-title
            >
            <v-card-text>This action cannot be undone.</v-card-text>
            <v-card-actions>
              <v-spacer />
              <v-btn
                text
                @click="
                  deleteDialog = false;
                  editDialog = true;
                "
              >
                Cancel
              </v-btn>
              <v-btn
                color="error"
                data-cy="deleteRoadConfirmButton"
                @click="
                  deleteDialog = false;
                  $emit('delete-road', tabRoad);
                  newRoadName = '';
                "
              >
                Delete
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
        <v-dialog
          v-model="addDialog"
          max-width="600"
          data-cy="addRoadDialog"
          @input="newRoadName = ''"
        >
          <v-card>
            <v-btn icon text style="float: right" @click="addDialog = false">
              <v-icon>mdi-close</v-icon>
            </v-btn>
            <v-card-title>Create Road</v-card-title>
            <v-card-text>
              <v-text-field
                v-if="addDialog"
                v-model="newRoadName"
                autofocus
                placeholder="New road name"
                data-cy="newRoadName"
                @keyup.enter="
                  if (validRoadName) {
                    createRoad();
                  }
                "
              />
              <v-row>
                <v-col cols="6">
                  <v-switch
                    v-model="duplicateRoad"
                    label="Duplicate existing"
                    data-cy="duplicateSwitch"
                  />
                </v-col>
                <v-col>
                  <v-select
                    v-model="duplicateRoadSource"
                    :disabled="!duplicateRoad"
                    :items="Object.keys(roads)"
                    data-cy="selectDuplicateRoadSource"
                    autocomplete
                  >
                    <template slot="item" slot-scope="{ item }">
                      <slot
                        :data-cy="'duplicateRoadSourceItem' + roads[item].id"
                        >{{ roads[item].name }}</slot
                      >
                    </template>
                    <template slot="selection" slot-scope="{ item }">
                      {{ roads[item].name }}
                    </template>
                  </v-select>
                </v-col>
              </v-row>
            </v-card-text>
            <v-card-actions>
              <v-spacer />
              <v-btn
                :disabled="!validRoadName"
                color="primary"
                data-cy="createRoadButton"
                @click="createRoad"
              >
                Create
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
      </v-tabs>
    </v-col>
    <v-col class="shrink">
      <v-btn
        type="submit"
        icon
        text
        background-color="primary"
        data-cy="addRoadButton"
        @click="addDialog = true"
      >
        <v-icon>mdi-plus</v-icon>
      </v-btn>
    </v-col>
  </v-row>
</template>

<script setup>
import { ref, computed, watch } from "vue";
import { useStore } from "../plugins/composition.js";

const store = useStore();

const emit = defineEmits(["delete-road", "add-road", "retrieve"]);

const addDialog = ref(false);
const deleteDialog = ref(false);
const editDialog = ref(false);
const duplicateRoad = ref(false);
const duplicateRoadSource = ref("$defaultroad$");
const newRoadName = ref("");
const tabRoad = ref(store.activeRoad);

const activeRoad = computed(() => store.activeRoad);
const roads = computed(() => store.roads);
const validRoadName = computed(() => {
  return !(otherRoadHasName("", newRoadName.value) || newRoadName.value === "");
});

watch(activeRoad, () => {
  tabRoad.value = activeRoad.value;
});

watch(store.unretrieved, () => {
  if (
    addDialog.value &&
    store.unretrieved.value.indexOf(duplicateRoadSource.value) === -1
  ) {
    addRoadFromDuplicate();
  }
});

const otherRoadHasName = (roadID, roadName) => {
  const otherRoadNames = Object.keys(roads.value).map((road) => {
    return road === roadID ? undefined : roads.value[road].name.toLowerCase();
  });
  return otherRoadNames.indexOf(roadName.toLowerCase()) >= 0;
};

const createRoad = () => {
  if (!duplicateRoad.value) {
    emit("add-road", newRoadName.value);
    addDialog.value = false;
    newRoadName.value = "";
  } else if (duplicateRoadSource.value in roads.value) {
    if (store.unretrieved.indexOf(duplicateRoadSource.value) >= 0) {
      emit("retrieve", duplicateRoadSource.value);
    } else {
      addRoadFromDuplicate();
    }
  }
};

const addRoadFromDuplicate = () => {
  emit(
    "add-road",
    newRoadName.value,
    roads.value[duplicateRoadSource.value].contents.coursesOfStudy.slice(0),
    roads.value[duplicateRoadSource.value].contents.selectedSubjects.map(
      (semester) => semester.slice(0),
    ),
    Object.assign(
      {},
      roads.value[duplicateRoadSource.value].contents.progressOverrides,
    ),
  );
  addDialog.value = false;
  newRoadName.value = "";
};

const renameRoad = () => {
  store.setRoadName({
    id: tabRoad.value,
    name: newRoadName.value,
  });
  editDialog.value = false;
  newRoadName.value = "";
};
</script>

<style>
/* CAREFUL! this is not scoped */
/*This is to prevent it from monopolizing all the space*/
div.v-tabs__container {
  display: unset;
  white-space: unset;
}
</style>
