<template>
  <!-- useful for adding dropdown: https://vuejs.org/v2/guide/forms.html -->

  <v-flex style="padding: 0px 18px 0px 18px; overflow: auto" data-cy="auditBox">
    <div style="display: flex; align-content: space-between; margin: 12px 0px">
      <v-autocomplete
        v-model="changeReqs"
        :hint="
          'Your Programs'.concat(
            changeReqs.length < 2 ? ' -- click to add a major' : '',
          )
        "
        :items="getCourses"
        item-text="medium-title"
        item-value="key"
        persistent-hint
        multiple
        chips
        deletable-chips
        no-data-text="'No results found'"
        data-cy="auditMajorChips"
      />
    </div>

    <v-treeview
      v-model="tree"
      :items="selectedTrees"
      item-key="uniqueKey"
      item-children="reqs"
      open-on-click
      dense
      transition
      :hoverable="false"
      :activatable="false"
    >
      <template slot="label" slot-scope="{ item, leaf }">
        <v-hover :disabled="!leaf || !canDrag(item)">
          <div
            slot-scope="{ hover }"
            class="req-container"
            :class="{
              'elevation-3': hover,
              'yellow lighten-3': isPetitioned(item) && !$vuetify.theme.dark,
              'yellow darken-3': isPetitioned(item) && $vuetify.theme.dark,
              'grey lighten-2': isIgnored(item) && !$vuetify.theme.dark,
              'grey darken-2': isIgnored(item) && $vuetify.theme.dark,
            }"
            :style="leaf && canDrag(item) ? 'cursor: grab' : 'cursor: pointer'"
            :data-cy="'auditItem' + item['list-id']"
          >
            <v-icon
              v-if="!('reqs' in item)"
              class="appendLeft"
              :style="fulfilledIcon(item)"
              @click="clickRequirement(item)"
            >
              {{
                item["plain-string"]
                  ? item["list-id"] in progressOverrides
                    ? item.fulfilled
                      ? "mdi-clipboard-check"
                      : "mdi-clipboard-text"
                    : "mdi-clipboard-alert"
                  : item.fulfilled
                    ? "mdi-check"
                    : "mdi-minus"
              }}
            </v-icon>
            <requirement
              :req="item"
              :is-leaf="leaf"
              @click.native="clickRequirement(item)"
              @click-info="reqInfo($event, item)"
              @click-petition="reqPetition($event, item)"
            />
          </div>
        </v-hover>
      </template>
    </v-treeview>

    <br />
    <p
      v-for="courseLink in getCourseLinks(selectedReqs)"
      :key="courseLink.link"
    >
      <a :href="courseLink.link">
        {{ courseLink.text }}
      </a>
    </p>
    <p>
      <a href="https://mit.turbovote.org/">Register to Vote!</a>
    </p>

    <v-dialog v-model="progressDialog" max-width="600" data-cy="progressDialog">
      <v-card v-if="progressReq !== undefined">
        <v-btn icon text style="float: right" @click="progressDialog = false">
          <v-icon>mdi-close</v-icon>
        </v-btn>
        <v-card-title>
          <h2>Manual Progress: {{ progressReq.title }}</h2>
        </v-card-title>
        <v-card-text>
          <p>
            Because this requirement is custom to the individual, you have to
            manually enter how many of these requirements you have completed for
            your degree.
          </p>
          <h3>
            {{ capitalize(progressReq.threshold.criterion) }} Completed:
            {{ newManualProgress }}/{{ progressReq.threshold.cutoff }}
          </h3>
          <v-layout justify-start style="width: 70%; margin: auto">
            <v-flex shrink style="width: 3em; margin-right: 1em">
              <v-text-field
                v-model="newManualProgress"
                type="number"
                @keyup.enter="updateManualProgress"
              />
            </v-flex>
            <v-flex>
              <v-slider
                v-model="newManualProgress"
                :max="progressReq.threshold.cutoff"
                :min="0"
                :step="1"
              />
            </v-flex>
          </v-layout>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn
            text
            @click="
              progressDialog = false;
              progressReq = undefined;
            "
          >
            Cancel
          </v-btn>
          <v-btn color="primary" @click="updateManualProgress">
            Update Progress
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog v-model="viewDialog" max-width="600" data-cy="auditViewDialog">
      <div v-if="dialogReq !== undefined">
        <v-card>
          <v-btn icon text style="float: right" @click="viewDialog = false">
            <v-icon>mdi-close</v-icon>
          </v-btn>
          <v-card-title>{{ dialogReq["title"] }}</v-card-title>
          <v-card-text v-if="'url' in dialogReq">
            <a target="_blank" :href="dialogReq['url']"
              >Link to the real, up to date requirements</a
            >
          </v-card-text>
          <v-card-text v-if="'desc' in dialogReq">
            {{ dialogReq["desc"] }}
          </v-card-text>
          <v-card-text>
            <v-progress-linear
              :value="dialogReq['percent_fulfilled']"
              height="25"
              :color="percentage(dialogReq)"
            >
              {{ dialogReq["percent_fulfilled"] }}% fulfilled
            </v-progress-linear>
          </v-card-text>
          <v-card-text v-if="'req' in dialogReq">
            {{ dialogReq["req"] }}
          </v-card-text>
          <v-card-text data-cy="viewDialogSatisfyingCourses">
            <b>Satisfying courses:</b>
            <div v-for="course in dialogReq['sat_courses']" :key="course">
              {{ course }}
            </div>
          </v-card-text>
          <v-card-actions>
            <v-spacer />
            <v-btn
              v-if="'title-no-degree' in dialogReq"
              color="error"
              data-cy="viewDialogRemoveButton"
              @click="
                deleteReq(dialogReq);
                viewDialog = false;
                dialogReq = undefined;
              "
            >
              <v-icon>mdi-delete</v-icon>
              Remove Requirement
            </v-btn>
          </v-card-actions>
        </v-card>
      </div>
    </v-dialog>

    <v-dialog v-model="petitionDialog" max-width="600">
      <v-card>
        <div v-if="petitionReq !== undefined">
          <v-btn
            icon
            text
            style="float: right"
            @click="
              petitionDialog = false;
              petitionReq = undefined;
            "
          >
            <v-icon>mdi-close</v-icon>
          </v-btn>
          <v-card-title v-if="'title' in petitionReq">
            <h2>Petition {{ petitionReq["title"] }}</h2>
          </v-card-title>
          <v-card-title v-else>
            <h2>Petition {{ petitionReq["req"] }}</h2>
          </v-card-title>
          <v-card-text
            v-if="reqPASubstitution !== undefined"
            class="petition-padding"
          >
            Requirement Petitioned by:
            <div v-for="course in reqPASubstitution" :key="course">
              {{ course }}
            </div>
          </v-card-text>
          <v-select
            v-model="petitionSelectCourses"
            :disabled="reqPAIgnore"
            :items="selectedSubjects.flat()"
            item-text="subject_id"
            label="Select Courses to Petition with:"
            no-data-text="No Courses Found"
            multiple
            chips
            autocomplete
            class="petition-padding"
          />
          <v-card-actions class="petition-padding">
            <v-checkbox v-model="reqPAIgnore" label="Ignore Petition" />
            <v-spacer />
            <v-btn
              color="success"
              :disabled="reqPAIgnore || petitionSelectCourses.length === 0"
              @click="submitPetition()"
            >
              Petition
            </v-btn>
            <v-btn
              color="error"
              :disabled="reqPAIgnore"
              @click="clearPetition()"
            >
              Reset Petition
            </v-btn>
          </v-card-actions>
        </div>
      </v-card>
    </v-dialog>
  </v-flex>
</template>

<script>
import Requirement from "./Requirement.vue";
import classInfoMixin from "../mixins/classInfo.js";
import courseLinksMixin from "./../mixins/courseLinks.js";
import { defineComponent } from "vue";

export default defineComponent({
  name: "AuditComponent",
  components: {
    requirement: Requirement,
  },
  mixins: [classInfoMixin, courseLinksMixin],
  props: {
    selectedSubjects: {
      type: Array,
      required: true,
    },
    selectedReqs: {
      type: Array,
      required: true,
    },
    reqTrees: {
      type: Object,
      required: true,
    },
    reqList: {
      type: Array,
      required: true,
    },
    progressOverrides: {
      type: Object,
      required: true,
    },
  },
  data: function () {
    return {
      tree: [],
      viewDialog: false,
      dialogReq: undefined,
      progressDialog: false,
      progressReq: undefined,
      petitionDialog: false,
      petitionReq: undefined,
      petitionSelectCourses: [],
      newManualProgress: 0,
    };
  },
  computed: {
    changeReqs: {
      get: function () {
        return this.selectedReqs;
      },
      set: function (newReqs) {
        const currentReqs = this.selectedReqs;
        if (currentReqs.length > newReqs.length) {
          const diff = currentReqs.find((x) => !newReqs.includes(x));
          this.$store.commit("removeReq", diff);
        } else {
          const newReq = newReqs[newReqs.length - 1];
          this.$store.commit("addReq", newReq);
        }
      },
    },
    getCourses: function () {
      const courses = this.reqList.slice(0);
      const sortKey = "medium-title";
      // NOTE: brute force way sorting the courses given the fields we have
      courses.sort(function (c1, c2) {
        const a = c1[sortKey].toLowerCase();
        const b = c2[sortKey].toLowerCase();
        if (
          (a.includes("major") && b.includes("major")) ||
          (a.includes("minor") && b.includes("minor"))
        ) {
          let n1 = a.split(" ")[0].split("-")[0];
          let n2 = b.split(" ")[0].split("-")[0];
          n1 = isNaN(n1) && !isNaN(n1.slice(0, -1)) ? n1.slice(0, -1) : n1;
          n2 = isNaN(n2) && !isNaN(n2.slice(0, -1)) ? n2.slice(0, -1) : n2;
          if (n1 === n2) return a.localeCompare(b);
          return (!isNaN(n1) && !isNaN(n2)) || (isNaN(n1) && isNaN(n2))
            ? n1 - n2
            : !isNaN(n1)
              ? -1
              : 1;
        } else if (a.includes("major") && b.includes("minor")) return -1;
        else if (b.includes("major") && a.includes("minor")) return 1;
        else if (a.includes("major") || a.includes("minor")) return -1;
        else if (b.includes("major") || b.includes("minor")) return 1;
        else return a.localeCompare(b);
      });
      return courses;
    },
    selectedTrees: function () {
      return this.selectedReqs.map(function (req, index) {
        if (req in this.reqTrees) {
          return this.assignListIDs(
            Object.assign({}, this.reqTrees[req]),
            index,
          );
        } else {
          return {
            title: "loading...",
            reqs: [],
            uniqueKey: index,
          };
        }
      }, this);
    },
    reqPASubstitution: {
      get: function () {
        const petitionReqPA =
          this.$store.state.roads[this.$store.state.activeRoad].contents
            .progressAssertions[this.petitionReq["list-id"]];
        // Checks if unique key in progressAssert, if it is, searches for substitution key
        if (petitionReqPA !== undefined) {
          return petitionReqPA.substitutions;
        } else {
          return undefined;
        }
      },
    },
    reqPAIgnore: {
      get: function () {
        const petitionReqPA =
          this.$store.state.roads[this.$store.state.activeRoad].contents
            .progressAssertions[this.petitionReq["list-id"]];
        if (petitionReqPA !== undefined) {
          return petitionReqPA.ignore;
        } else {
          return false; // So checkbox properly updates when resetPetition is used
        }
      },
      set: function (ignoreVal) {
        this.$store.commit("setPAIgnore", {
          uniqueKey: this.petitionReq["list-id"],
          isIgnored: ignoreVal,
        });
      },
    },
  },
  methods: {
    fulfilledIcon: function (req) {
      return req.fulfilled &&
        (req.req !== undefined || req.sat_courses.length > 0)
        ? "color: #00b300;"
        : "";
    },
    reqInfo: function (event, req) {
      event.preventDefault();
      event.stopPropagation();
      this.viewDialog = true;
      this.dialogReq = req;
    },
    reqPetition: function (event, req) {
      event.preventDefault();
      event.stopPropagation();
      this.petitionDialog = true;
      this.petitionReq = req;
    },
    clickRequirement: function (item) {
      if (item.req !== undefined) {
        if (!item["plain-string"]) {
          let usedReq = item.req;
          if (usedReq.indexOf("GIR:") === 0) {
            usedReq = usedReq.substring(4);
          }
          this.$store.commit("pushClassStack", usedReq);
        } else {
          this.startProgressDialog(item);
        }
      }
    },
    // gives each list and sublist an id
    // progress overrides are a dictionary where the keys are these list ids and the values are the manual progress
    // for example, the 3rd requirement of the 1st requirement of GIRs (CAL1) would have id gir.0.2
    assignListIDs: function (req, index) {
      if ("reqs" in req && "list-id" in req) {
        let currentListID = req["list-id"];
        if (currentListID.indexOf(".reql") >= 0) {
          // if the requirement is top level, it will have .reql at the end and this needs to be removed
          req["list-id"] = req["list-id"].substring(
            0,
            req["list-id"].indexOf(".reql"),
          );
          currentListID = req["list-id"];
        }
        req.uniqueKey = index + "-" + req["list-id"];
        for (let r = 0; r < req.reqs.length; r++) {
          // give each sub-requirement a list id of [parent list id].[index]
          Object.assign(req.reqs[r], { "list-id": currentListID + "." + r });
          // assign list ids to each of the children
          req.reqs[r] = this.assignListIDs(req.reqs[r], index);
        }
      }
      return req;
    },
    startProgressDialog: function (req) {
      this.progressReq = Object.assign(
        { threshold: { criterion: "subject", cutoff: 1, type: "GTE" } },
        req,
      );
      this.progressDialog = true;
      if (this.progressReq["list-id"] in this.progressOverrides) {
        this.newManualProgress =
          this.progressOverrides[this.progressReq["list-id"]];
      }
    },
    capitalize: function (word) {
      return word[0].toUpperCase() + word.substring(1);
    },
    updateManualProgress: function () {
      if (this.progressReq["list-id"] !== undefined) {
        this.$store.commit("updateProgress", {
          listID: this.progressReq["list-id"],
          progress: this.newManualProgress,
        });
      }
      this.progressReq = undefined;
      this.progressDialog = false;
      this.newManualProgress = 0;
    },
    percentage: function (req) {
      return req.fulfilled
        ? "#00b300"
        : req.percent_fulfilled > 15
          ? "#efce15"
          : "#ef8214";
    },
    deleteReq: function (req) {
      const reqName = req["list-id"];
      this.$store.commit("removeReq", reqName);
    },
    submitPetition: function () {
      this.$store.commit("setPASubstitutions", {
        uniqueKey: this.petitionReq["list-id"],
        newReqs: this.petitionSelectCourses,
      });
      this.petitionSelectCourses = [];
    },
    clearPetition: function () {
      this.$store.commit(
        "removeProgressAssertion",
        this.petitionReq["list-id"],
      );
    },
    isPetitioned: function (req) {
      if (
        req["list-id"] in
        this.$store.state.roads[this.$store.state.activeRoad].contents
          .progressAssertions
      ) {
        return !(
          "ignore" in
          this.$store.state.roads[this.$store.state.activeRoad].contents
            .progressAssertions[req["list-id"]]
        );
      } else {
        return false;
      }
    },
    isIgnored: function (req) {
      if (
        req["list-id"] in
        this.$store.state.roads[this.$store.state.activeRoad].contents
          .progressAssertions
      ) {
        return (
          "ignore" in
          this.$store.state.roads[this.$store.state.activeRoad].contents
            .progressAssertions[req["list-id"]]
        );
      } else {
        return false;
      }
    },
  },
});
</script>

<style scoped>
.appendLeft {
  float: left;
  position: relative;
  align-self: center;
}
.req-container {
  display: flex;
}
.petition-padding {
  padding-left: 5%;
  padding-right: 5%;
}
</style>
<style>
/* Makes the leaf margin consistent with the margin of expandable reqs */
div.v-treeview-node.v-treeview-node--leaf {
  margin-left: 26px;
}
</style>
