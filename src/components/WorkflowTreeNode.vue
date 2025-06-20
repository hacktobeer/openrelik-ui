<!--
Copyright 2024 Google LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
<template>
  <li>
    <v-dialog v-model="showTaskConfigForm" width="600">
      <v-card width="600" class="mx-auto">
        <v-card-title>Config for {{ node.display_name }}</v-card-title>
        <v-card-text>
          <task-config-form
            :fields="node.task_config"
            @save="saveTaskConfig($event)"
            @cancel="showTaskConfigForm = false"
          ></task-config-form>
        </v-card-text>
      </v-card>
    </v-dialog>
    <span
      class="node-content"
      :class="
        $vuetify.theme.name === 'dark'
          ? 'dark-grey-background'
          : 'white-background'
      "
    >
      <v-menu
        v-if="celeryTask.status_short"
        activator="parent"
        location="bottom"
        :close-on-content-click="false"
      >
        <v-card width="600px" class="pa-4 mt-2">
          <task-result-default
            :worker="workflow"
            :task="celeryTask"
          ></task-result-default>
        </v-card>
      </v-menu>
      <template v-if="node.isRoot">
        <div>
          <div
            v-for="(file, index) in showAllFiles
              ? workflow.files
              : workflow.files.slice(0, 10)"
            :key="file.id"
            :class="file.is_deleted ? 'red-text' : ''"
            :title="file.display_name"
            class="file-container"
          >
            <v-icon
              v-if="file.data_type.startsWith('cloud:')"
              class="mr-2 mt-n1"
              >mdi-cloud-outline</v-icon
            >
            <v-icon v-else size="small" class="mr-1" style="margin-top: -3px"
              >mdi-file-outline</v-icon
            >
            <div class="truncate">
              <span style="font-size: 0.9em">{{ file.display_name }}</span>
            </div>

            <v-btn
              v-if="!workflow.tasks.length && workflow.files.length === 1"
              icon
              variant="text"
              size="x-small"
              class="ml-1"
            >
              <v-icon color="grey-lighten-1" size="x-large">mdi-plus</v-icon>
            </v-btn>
          </div>

          <div
            v-if="workflow.files.length > 10"
            class="mt-1"
            style="
              text-decoration: underline;
              cursor: pointer;
              font-size: 0.9em;
            "
            @click="showAllFiles = !showAllFiles"
          >
            {{
              showAllFiles
                ? "Show less"
                : `Show all ${workflow.files.length} files`
            }}
          </div>
        </div>
      </template>
      <template v-else>
        <div>
          <task-status-icon
            class="mr-1"
            :task-status="celeryTask.status_short"
          ></task-status-icon>
          {{ node.display_name }}
          <span v-if="celeryTask.runtime">
            <small class="">({{ celeryTask.runtime.toFixed(1) }}s)</small>
          </span>
          <div v-if="celeryTask.file_reports && celeryTask.file_reports.length">
            <small class="ml-1">
              Reports:
              <strong>{{ celeryTask.file_reports.length }}</strong> reports
            </small>
          </div>
          <div
            v-if="
              hasHighPriorityFileReports && hasHighPriorityFileReports.length
            "
          >
            <small class="ml-1 red-text">
              <strong>{{ hasHighPriorityFileReports.length }}</strong> report(s)
              with high priority findings
            </small>
          </div>
          <div v-if="hasHighPriorityTaskReport">
            <small class="ml-1 red-text"> High priority findings </small>
          </div>

          <span v-if="hasTaskConfig">
            <v-btn
              v-if="!Object.keys(celeryTask).length"
              icon
              variant="text"
              size="x-small"
              @click.stop="showTaskConfigForm = true"
            >
              <v-icon>mdi-cog-outline</v-icon>
            </v-btn>
          </span>
          <v-btn
            v-if="!workflow.tasks.length"
            icon
            variant="text"
            size="x-small"
            class="ml-1"
            @click="removeTask(node)"
          >
            <v-icon color="grey-lighten-1" size="x-large"
              >mdi-trash-can-outline</v-icon
            >
          </v-btn>

          <v-btn
            v-if="!workflow.tasks.length"
            icon
            variant="text"
            size="x-small"
            class="ml-1"
          >
            <v-icon color="grey-lighten-1" size="x-large">mdi-plus</v-icon>
          </v-btn>
        </div>
        <div class="ml-1" v-for="(option, optionIndex) in node.task_config">
          <small
            v-if="
              option.hasOwnProperty('value') &&
              option.value !== null &&
              option.value !== undefined &&
              option.value !== ''
            "
          >
            <strong>{{ option.name }}: </strong>
            <span v-if="Array.isArray(option.value)">
              <div
                v-for="(value, index) in showOption[optionIndex]
                  ? option.value
                  : option.value.slice(0, 5)"
                :key="index"
              >
                {{ value }}
              </div>
              <div
                v-if="option.value.length > 5"
                class="mt-1"
                style="
                  text-decoration: underline;
                  cursor: pointer;
                  font-size: 0.9em;
                "
                @click.stop="showOption[optionIndex] = !showOption[optionIndex]"
              >
                {{
                  showOption[optionIndex]
                    ? "See less"
                    : `See all ${option.value.length}`
                }}
              </div>
            </span>
            <span v-else>{{ option.value }} </span>
          </small>
        </div>
      </template>

      <workflow-task-dropdown
        v-if="!workflow.tasks.length"
        :isRootNode="node.isRoot"
        @add-task="addTask($event, node)"
        @remove-task="removeTask(node)"
      ></workflow-task-dropdown>
    </span>
    <ul v-if="node.tasks && node.tasks.length">
      <tree-node
        v-for="task in node.tasks"
        :node="task"
        :workflow="workflow"
        :add-task="addTask"
        :remove-task="removeTask"
        :update-workflow="updateWorkflow"
      ></tree-node>
    </ul>
  </li>
</template>

<script>
import WorkflowTaskDropdown from "./WorkflowTaskDropdown";
import TaskStatusIcon from "./TaskStatusIcon";
import TaskResultDefault from "@/components/TaskResultDefault.vue";
import TaskConfigForm from "./TaskConfigForm.vue";

export default {
  name: "treeNode",
  props: {
    node: Object,
    workflow: Object,
    addTask: Function,
    removeTask: Function,
    updateWorkflow: Function,
  },
  components: {
    WorkflowTaskDropdown,
    TaskStatusIcon,
    TaskResultDefault,
    TaskConfigForm: TaskConfigForm,
  },
  data() {
    return {
      showTaskConfigForm: false,
      showAllFiles: false,
      showOption: {},
    };
  },
  computed: {
    celeryTask() {
      return (
        this.workflow.tasks.filter((task) => task.uuid === this.node.uuid)[0] ||
        {}
      );
    },
    hasTaskConfig() {
      return this.node.task_config && this.node.task_config.length;
    },
    hasTaskConfigWithValue() {
      return this.node.task_config.some((option) => {
        return (
          option.hasOwnProperty("value") &&
          option.value !== null &&
          option.value !== undefined &&
          option.value !== ""
        );
      });
    },
    hasHighPriorityFileReports() {
      if (
        !this.celeryTask.file_reports ||
        !this.celeryTask.file_reports.length
      ) {
        return false;
      }
      return this.celeryTask.file_reports.filter(
        (report) => report.priority >= 40
      );
    },
    hasHighPriorityTaskReport() {
      if (!this.celeryTask.task_report) {
        return false;
      }
      return this.celeryTask.task_report.priority >= 40;
    },
    nodeIcon() {
      return this.celeryTask.status_short === "PROGRESS"
        ? "mdi-chart-box-outline"
        : "mdi-information-outline";
    },
  },
  methods: {
    saveTaskConfig(formData) {
      // Loop through the task options in the object
      this.node.task_config.forEach((option) => {
        // Check if there's a corresponding value in the formData
        if (formData.hasOwnProperty(option.name)) {
          // Update the option with the value from the formData
          option.value = formData[option.name];
        }
      });
      this.updateWorkflow();
      this.showTaskConfigForm = false;
    },
  },
};
</script>

<style scoped>
.red-text {
  color: red;
}
.file-container {
  display: flex;
  align-items: center;
  width: 100%;
}
.truncate {
  flex-grow: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
</style>
