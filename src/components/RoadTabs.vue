<template>
  <v-layout>
    <v-tabs v-model="tabRoad" background-color="transparent" show-arrows>
      <v-tabs-slider />
      <v-tab
        v-for="roadId in Object.keys(roads)"
        :key="roadId"
        :href="`#${roadId}`"
        :data-cy="'roadTab' + roadId"
        @click="$store.commit('setActiveRoad', roadId)"
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
            <v-layout>
              <v-flex xs6>
                <v-switch
                  v-model="duplicateRoad"
                  label="Duplicate existing"
                  data-cy="duplicateSwitch"
                />
              </v-flex>
              <v-flex>
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
              </v-flex>
            </v-layout>
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
    <v-flex>
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
    </v-flex>
  </v-layout>
</template>

<script>
import { defineComponent } from "vue";

export default defineComponent({
  name: "RoadTabs",
  components: {},
  data: function () {
    return {
      addDialog: false,
      deleteDialog: false,
      editDialog: false,
      duplicateRoad: false,
      duplicateRoadSource: "$defaultroad$",
      newRoadName: "",
      tabRoad: this.$store.state.activeRoad,
    };
  },
  computed: {
    activeRoad() {
      return this.$store.state.activeRoad;
    },
    roads() {
      return this.$store.state.roads;
    },
    validRoadName: function () {
      return !(
        this.otherRoadHasName("", this.newRoadName) || this.newRoadName === ""
      );
    },
  },
  watch: {
    activeRoad: function () {
      this.tabRoad = this.activeRoad;
    },
    "$store.state.unretrieved": function () {
      if (
        this.addDialog &&
        this.$store.state.unretrieved.indexOf(this.duplicateRoadSource) === -1
      ) {
        this.addRoadFromDuplicate();
      }
    },
  },
  methods: {
    otherRoadHasName: function (roadID, roadName) {
      const otherRoadNames = Object.keys(this.roads).map(
        function (road) {
          return road === roadID
            ? undefined
            : this.roads[road].name.toLowerCase();
        }.bind(this),
      );
      return otherRoadNames.indexOf(roadName.toLowerCase()) >= 0;
    },
    createRoad: function () {
      if (!this.duplicateRoad) {
        this.$emit("add-road", this.newRoadName);
        this.addDialog = false;
        this.newRoadName = "";
      } else if (this.duplicateRoadSource in this.roads) {
        if (
          this.$store.state.unretrieved.indexOf(this.duplicateRoadSource) >= 0
        ) {
          this.$emit("retrieve", this.duplicateRoadSource);
        } else {
          this.addRoadFromDuplicate();
        }
      }
    },
    addRoadFromDuplicate: function () {
      this.$emit(
        "add-road",
        this.newRoadName,
        this.roads[this.duplicateRoadSource].contents.coursesOfStudy.slice(0),
        this.roads[this.duplicateRoadSource].contents.selectedSubjects.map(
          (semester) => semester.slice(0),
        ),
        Object.assign(
          {},
          this.roads[this.duplicateRoadSource].contents.progressOverrides,
        ),
      );
      this.addDialog = false;
      this.newRoadName = "";
    },
    renameRoad: function () {
      this.$store.commit("setRoadName", {
        id: this.tabRoad,
        name: this.newRoadName,
      });
      this.editDialog = false;
      this.newRoadName = "";
    },
  },
});
</script>

<style>
/* CAREFUL! this is not scoped */
/*This is to prevent it from monopolizing all the space*/
div.v-tabs__container {
  display: unset;
  white-space: unset;
}
</style>
