<template>
  <div v-if="!selectedAgentPk">No agent selected</div>
  <div v-else-if="Object.keys(sortedUpdates).length === 0">No Patches</div>
  <div v-else class="q-pa-xs">
    <q-btn dense flat push @click="refreshUpdates(updates.pk)" icon="refresh" class="q-mr-sm" />
    <span v-if="summary.patches_last_installed" class="text-bold">
      Patches last installed: {{ summary.patches_last_installed }}
    </span>
    <span v-else class="text-bold">Patches last installed: Never</span>
    <q-table
      dense
      :table-class="{ 'table-bgcolor': !$q.dark.isActive, 'table-bgcolor-dark': $q.dark.isActive }"
      class="tabs-tbl-sticky"
      :style="{ 'max-height': tabsTableHeight }"
      :rows="sortedUpdates"
      :columns="columns"
      :visible-columns="visibleColumns"
      v-model:pagination="pagination"
      :filter="filter"
      row-key="id"
      binary-state-sort
      hide-bottom
      virtual-scroll
      :rows-per-page-options="[0]"
    >
      <template v-slot:body="props">
        <q-tr :props="props">
          <q-menu context-menu>
            <q-list dense style="min-width: 100px">
              <q-item v-if="!props.row.installed" clickable v-close-popup @click="editPolicy(props.row.id, 'inherit')">
                <q-item-section>Inherit</q-item-section>
              </q-item>
              <q-item v-if="!props.row.installed" clickable v-close-popup @click="editPolicy(props.row.id, 'approve')">
                <q-item-section>Approve</q-item-section>
              </q-item>
              <q-item v-if="!props.row.installed" clickable v-close-popup @click="editPolicy(props.row.id, 'ignore')">
                <q-item-section>Ignore</q-item-section>
              </q-item>
              <q-item v-if="!props.row.installed" clickable v-close-popup @click="editPolicy(props.row.id, 'nothing')">
                <q-item-section>Do Nothing</q-item-section>
              </q-item>
            </q-list>
          </q-menu>
          <!-- policy -->
          <q-td>
            <q-icon v-if="props.row.action === 'nothing'" name="fiber_manual_record" color="grey">
              <q-tooltip>Do Nothing</q-tooltip>
            </q-icon>
            <q-icon v-else-if="props.row.action === 'approve'" name="fas fa-check" color="primary">
              <q-tooltip>Approve</q-tooltip>
            </q-icon>
            <q-icon v-else-if="props.row.action === 'ignore'" name="fas fa-check" color="negative">
              <q-tooltip>Ignore</q-tooltip>
            </q-icon>
            <q-icon v-else-if="props.row.action === 'inherit'" name="fiber_manual_record" color="accent">
              <q-tooltip>Inherit</q-tooltip>
            </q-icon>
          </q-td>
          <q-td>
            <q-icon v-if="props.row.installed" name="fas fa-check" color="positive">
              <q-tooltip>Installed</q-tooltip>
            </q-icon>
            <q-icon v-else-if="props.row.action == 'approve'" name="fas fa-tasks" color="primary">
              <q-tooltip>Pending</q-tooltip>
            </q-icon>
            <q-icon v-else-if="props.row.action == 'ignore'" name="fas fa-ban" color="negative">
              <q-tooltip>Ignored</q-tooltip>
            </q-icon>
            <q-icon v-else name="fas fa-exclamation" color="warning">
              <q-tooltip>Missing</q-tooltip>
            </q-icon>
          </q-td>
          <q-td>{{ formatSeverity(props.row.severity) }}</q-td>
          <q-td>{{ formatMessage(props.row.title) }}</q-td>
          <q-td
            @click="showFullMsg(props.row.title, props.row.description, props.row.more_info_urls, props.row.categories)"
          >
            <span style="cursor: pointer; text-decoration: underline" class="text-primary">{{
              formatMessage(props.row.description)
            }}</span>
          </q-td>
          <q-td>{{ formatDjangoDate(props.row.date_installed) }}</q-td>
        </q-tr>
      </template>
    </q-table>
  </div>
</template>

<script>
import { mapGetters, mapState } from "vuex";
import mixins from "@/mixins/mixins";

export default {
  name: "WindowsUpdates",
  mixins: [mixins],
  data() {
    return {
      filter: "",
      pagination: {
        rowsPerPage: 0,
        sortBy: "installed",
        descending: false,
      },
      columns: [
        { name: "id", label: "ID", field: "id" },
        {
          name: "action",
          field: "action",
          align: "left",
        },
        {
          name: "installed",
          field: "installed",
          align: "left",
        },
        {
          name: "severity",
          label: "Severity",
          field: "severity",
          align: "left",
          sortable: true,
        },
        {
          name: "title",
          label: "Name",
          field: "title",
          align: "left",
          sortable: true,
        },
        {
          name: "description",
          label: "More Info",
          field: "description",
          align: "left",
          sortable: true,
        },
        {
          name: "date_installed",
          label: "Installed On",
          field: "date_installed",
          align: "left",
          sortable: true,
          sort: (a, b) => this.dateStringToUnix(a) - this.dateStringToUnix(b),
        },
        {
          name: "more_info_urls",
          field: "more_info_urls",
        },
        {
          name: "categories",
          field: "categories",
        },
      ],
      visibleColumns: ["action", "installed", "severity", "title", "description", "date_installed"],
    };
  },
  methods: {
    formatSeverity(severity) {
      return !severity ? "Other" : severity;
    },
    editPolicy(pk, policy) {
      const data = { pk: pk, policy: policy };
      this.$axios
        .patch(`/winupdate/editpolicy/`, data)
        .then(r => {
          this.refreshUpdates(this.updates.pk);
          this.notifySuccess("Policy edited!");
        })
        .catch(e => {});
    },
    refreshUpdates(pk) {
      this.$store.dispatch("loadWinUpdates", pk);
    },
    formatMessage(msg) {
      return msg.substring(0, 80) + "...";
    },
    showFullMsg(title, msg, urls, categories) {
      let support_urls = "";
      urls.forEach(u => {
        support_urls += `<a href='${u}' target='_blank'>${u}</a><br/>`;
      });
      let cats = categories.join(", ");
      this.$q.dialog({
        title: title,
        message:
          `<b>Categories:</b> ${cats}<br/><br/>` +
          "<b>Description</b><br/>" +
          msg.split(". ").join(".<br />") +
          `<br/><br/><b>Support Urls</b><br/>${support_urls}`,
        html: true,
        fullWidth: true,
      });
    },
  },
  computed: {
    summary() {
      return this.$store.state.agentSummary;
    },
    ...mapState({
      updates: state => Object.freeze(state.winUpdates),
    }),
    ...mapGetters(["sortedUpdates", "selectedAgentPk", "tabsTableHeight"]),
  },
};
</script>