<template>
  <div>
    <gov-heading size="l">Manage reports</gov-heading>

    <ck-loader v-if="loadingReportSchedules" />
    <section
      v-else
      v-for="(reportType, index) in reportTypes"
      :key="reportType.type"
    >
      <gov-inset-text>
        <gov-heading size="m">{{ reportType.type }}</gov-heading>
        <gov-body>{{ reportType.description }}</gov-body>

        <gov-heading size="s">Scheduled generation</gov-heading>
        <gov-body
          >This decides the regularity of when reports are sent to
          you.</gov-body
        >
        <ck-radio-input
          :id="`repeat_type[${reportType.type}]`"
          v-model="reportType.scheduleForm.repeat_type"
          :error="reportType.scheduleForm.$errors.get('repeat_type')"
          @input="reportType.scheduleForm.$errors.clear('repeat_type')"
          :options="repeatTypeOptions"
        />
        <gov-button
          v-if="!reportType.scheduleForm.$submitting"
          type="submit"
          @click="onSaveReportSchedule(reportType)"
          >Save frequency</gov-button
        >
        <gov-button v-else type="submit" disabled>Saving...</gov-button>

        <gov-heading size="s">Generate a report now</gov-heading>
        <gov-body
          >This allows you to generate a one off report which will begin
          downloading immediately.</gov-body
        >
        <template v-if="acceptsDateRange(reportType)">
          <ck-date-input
            :id="`starts_at[${reportType.type}]`"
            v-model="reportType.generateForm.starts_at"
            :error="reportType.generateForm.$errors.get('starts_at')"
            @input="reportType.generateForm.$errors.clear('starts_at')"
            label="From date"
          />
          <ck-date-input
            :id="`ends_at[${reportType.type}]`"
            v-model="reportType.generateForm.ends_at"
            :error="reportType.generateForm.$errors.get('ends_at')"
            @input="reportType.generateForm.$errors.clear('ends_at')"
            label="To date"
          />
        </template>
        <gov-button
          v-if="!reportType.generateForm.$submitting"
          type="submit"
          @click="onGenerate(reportType)"
          >Generate and download</gov-button
        >
        <gov-button v-else type="submit" disabled>Generating...</gov-button>
      </gov-inset-text>
      <gov-section-break
        v-if="index < reportTypes.length - 1"
        size="l"
        visible
      />
    </section>
  </div>
</template>

<script>
import http from "@/http";
import Form from "@/classes/Form";
import CkDateInput from "@/components/Ck/CkDateInput";

export default {
  name: "EditReportsPage",
  components: { CkDateInput },
  data() {
    return {
      loadingReportSchedules: false,
      reportTypes: [
        {
          type: "Audit Logs Export",
          description:
            "Generate a report of user and visitor activity from the audit " +
            "logs between certain times.",
          scheduleForm: new Form({
            report_type: "Audit Logs Export",
            repeat_type: null
          }),
          generateForm: new Form({
            report_type: "Audit Logs Export",
            starts_at: "",
            ends_at: ""
          })
        },
        {
          type: "Feedback Export",
          description:
            "Generate a report of all the feedback posted to the site using " +
            "the inbuilt feedback feature between certain times. ",
          scheduleForm: new Form({
            report_type: "Feedback Export",
            repeat_type: null
          }),
          generateForm: new Form({
            report_type: "Feedback Export",
            starts_at: "",
            ends_at: ""
          })
        },
        {
          type: "Locations Export",
          description: `Generate a report with all the locations of services on ${this.appName}, and the number of services delivered at each.`,
          scheduleForm: new Form({
            report_type: "Locations Export",
            repeat_type: null
          }),
          generateForm: new Form({
            report_type: "Locations Export",
            starts_at: "",
            ends_at: ""
          })
        },
        {
          type: "Organisations Export",
          description: `Generate a report of all the organisations on ${this.appName} with the number of services and attributed accounts.`,
          scheduleForm: new Form({
            report_type: "Organisations Export",
            repeat_type: null
          }),
          generateForm: new Form({
            report_type: "Organisations Export",
            starts_at: "",
            ends_at: ""
          })
        },
        {
          type: "Referrals Export",
          description:
            "Generate a report of all referrals between a certain time " +
            "period. The report contains no personal information.",
          scheduleForm: new Form({
            report_type: "Referrals Export",
            repeat_type: null
          }),
          generateForm: new Form({
            report_type: "Referrals Export",
            starts_at: "",
            ends_at: ""
          })
        },
        {
          type: "Search Histories Export",
          description:
            "Generate a report of all the keyword search queries entered on " +
            "the site, between a certain time period.",
          scheduleForm: new Form({
            report_type: "Search Histories Export",
            repeat_type: null
          }),
          generateForm: new Form({
            report_type: "Search Histories Export",
            starts_at: "",
            ends_at: ""
          })
        },
        {
          type: "Services Export",
          description:
            "Generate a list of all services, including contact " +
            "details, whether they have referrals enabled, the last time " +
            "updated, contact details etc.",
          scheduleForm: new Form({
            report_type: "Services Export",
            repeat_type: null
          }),
          generateForm: new Form({
            report_type: "Services Export",
            starts_at: "",
            ends_at: ""
          })
        },
        {
          type: "Users Export",
          description: `Generate a report of all users on the site, contact details, and last login times.`,
          scheduleForm: new Form({
            report_type: "Users Export",
            repeat_type: null
          }),
          generateForm: new Form({
            report_type: "Users Export",
            starts_at: "",
            ends_at: ""
          })
        },
        {
          type: "Historic Update Requests Export",
          description: `Generate a report of all approved/rejected update requests.`,
          scheduleForm: new Form({
            report_type: "Historic Update Requests Export",
            repeat_type: null
          }),
          generateForm: new Form({
            report_type: "Historic Update Requests Export",
            starts_at: "",
            ends_at: ""
          })
        }
      ],
      repeatTypeOptions: [
        { label: "Not scheduled", value: null },
        { label: "Weekly", value: "weekly" },
        { label: "Monthly", value: "monthly" }
      ]
    };
  },
  methods: {
    async fetchReportSchedules() {
      this.loadingReportSchedules = true;

      const reportSchedules = await this.fetchAll("/report-schedules");
      reportSchedules.forEach(reportSchedule => {
        const reportType = this.reportTypes.find(
          reportType => reportType.type === reportSchedule.report_type
        );

        reportType.scheduleForm = new Form(reportSchedule);
      });

      this.loadingReportSchedules = false;
    },
    async onGenerate(reportType) {
      const data = await reportType.generateForm.post("/reports");
      const reportId = data.data.id;
      const file = await http.get(`/reports/${reportId}/download`);

      const regex = /filename[^;=\n]*=(?:(\\?['"])(.*?)\1|(?:[^\s]+'.*?')?([^;\n]*))/;
      const filename = regex.exec(
        file.request.getResponseHeader("Content-Disposition")
      )[2];
      const url = window.URL.createObjectURL(new Blob([file.data]));
      const link = document.createElement("a");
      link.href = url;
      link.setAttribute("download", filename);
      document.body.appendChild(link);
      link.click();
    },
    async onSaveReportSchedule(reportType) {
      if (reportType.scheduleForm.hasOwnProperty("id")) {
        // Update or delete.

        // If delete.
        if (reportType.scheduleForm.repeat_type === null) {
          await reportType.scheduleForm.delete(
            `/report-schedules/${reportType.scheduleForm.id}`
          );
          reportType.scheduleForm = new Form({
            report_type: reportType.scheduleForm.report_type,
            repeat_type: null
          });

          return;
        }

        // Else update.
        const data = await reportType.scheduleForm.put(
          `/report-schedules/${reportType.scheduleForm.id}`
        );
        reportType.scheduleForm = new Form(data.data);
      } else {
        // Create.

        // Skip if repeat type is still null.
        if (reportType.scheduleForm.repeat_type === null) {
          return;
        }

        const data = await reportType.scheduleForm.post("/report-schedules");
        reportType.scheduleForm = new Form(data.data);
      }
    },
    acceptsDateRange(reportType) {
      switch (reportType.type) {
        case "Audit Logs Export":
          return true;
        case "Feedback Export":
          return true;
        case "Locations Export":
          return false;
        case "Organisations Export":
          return false;
        case "Referrals Export":
          return true;
        case "Search Histories Export":
          return true;
        case "Services Export":
          return false;
        case "Users Export":
          return false;
        case "Historic Update Requests Export":
          return true;
      }
    }
  },
  created() {
    this.fetchReportSchedules();
  }
};
</script>
