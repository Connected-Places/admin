<template>
  <ck-loader v-if="loading" />
  <div v-else>
    <gov-body>
      For service location
      <gov-link
        :to="{
          name: 'service-locations-show',
          params: { serviceLocation: original.id }
        }"
        >{{ service.name }} at {{ location.address_line_1 }}</gov-link
      >.
    </gov-body>

    <gov-table>
      <template slot="body">
        <gov-table-row>
          <gov-table-header scope="column"></gov-table-header>
          <gov-table-header scope="column">From</gov-table-header>
          <gov-table-header scope="column">To</gov-table-header>
        </gov-table-row>

        <gov-table-row v-if="serviceLocation.name">
          <gov-table-header top scope="row">Name</gov-table-header>
          <gov-table-cell>{{ original.name }}</gov-table-cell>
          <gov-table-cell>{{ serviceLocation.name }}</gov-table-cell>
        </gov-table-row>

        <gov-table-row
          v-if="serviceLocation.hasOwnProperty('regular_opening_hours')"
        >
          <gov-table-header top scope="row"
            >Regular opening hours</gov-table-header
          >
          <gov-table-cell>
            <gov-list v-if="original.regular_opening_hours.length > 0">
              <li
                v-for="(regularOpeningHour,
                index) in original.regular_opening_hours"
                :key="index"
                v-text="formatRegularOpeningHour(regularOpeningHour)"
              />
            </gov-list>
            <template v-else>None</template>
          </gov-table-cell>
          <gov-table-cell>
            <gov-list v-if="serviceLocation.regular_opening_hours.length > 0">
              <li
                v-for="(regularOpeningHour,
                index) in serviceLocation.regular_opening_hours"
                :key="index"
                v-text="formatRegularOpeningHour(regularOpeningHour)"
              />
            </gov-list>
            <template v-else>None</template>
          </gov-table-cell>
        </gov-table-row>

        <gov-table-row
          v-if="serviceLocation.hasOwnProperty('holiday_opening_hours')"
        >
          <gov-table-header top scope="row"
            >Holiday opening hours</gov-table-header
          >
          <gov-table-cell>
            <gov-list v-if="original.holiday_opening_hours.length > 0">
              <li
                v-for="(holidayOpeningHour,
                index) in original.holiday_opening_hours"
                :key="index"
                v-text="formatHolidayOpeningHour(holidayOpeningHour)"
              />
            </gov-list>
            <template v-else>None</template>
          </gov-table-cell>
          <gov-table-cell>
            <gov-list v-if="serviceLocation.holiday_opening_hours.length > 0">
              <li
                v-for="(holidayOpeningHour,
                index) in serviceLocation.holiday_opening_hours"
                :key="index"
                v-text="formatHolidayOpeningHour(holidayOpeningHour)"
              />
            </gov-list>
            <template v-else>None</template>
          </gov-table-cell>
        </gov-table-row>

        <gov-table-row v-if="serviceLocation.hasOwnProperty('image_file_id')">
          <gov-table-header top scope="row">Image</gov-table-header>
          <gov-table-cell>
            <ck-image v-if="original.image" :file-id="original.image.id" />
          </gov-table-cell>
          <gov-table-cell>
            <ck-image
              v-if="serviceLocation.image_file_id"
              :file-id="serviceLocation.image_file_id"
            />
            <img
              v-else
              :src="
                apiUrl(
                  `/service-locations/${serviceLocation.id}/image.png?update_request_id=${updateRequestId}`
                )
              "
              alt="Service location image"
              class="ck-logo"
            />
          </gov-table-cell>
        </gov-table-row>
      </template>
    </gov-table>
  </div>
</template>

<script>
import moment from "moment";
import http from "@/http";
import CkImage from "@/components/Ck/CkImage";

export default {
  name: "ServiceLocationDetails",
  components: { CkImage },
  props: {
    updateRequestId: {
      required: true,
      type: String
    },

    requestedAt: {
      required: true,
      type: String
    },

    serviceLocation: {
      required: true,
      type: Object
    }
  },
  data() {
    return {
      loading: false,
      original: null,
      service: null,
      location: null
    };
  },
  methods: {
    async fetchAll() {
      this.loading = true;

      await this.fetchOriginal();
      await this.fetchService();
      await this.fetchLocation();

      this.loading = false;
    },
    async fetchOriginal() {
      const { data } = await http.get(
        `/service-locations/${this.serviceLocation.id}`
      );
      this.original = data.data;
    },
    async fetchService() {
      const { data } = await http.get(`/services/${this.original.service_id}`);
      this.service = data.data;
    },
    async fetchLocation() {
      const { data } = await http.get(
        `/locations/${this.original.location_id}`
      );
      this.location = data.data;
    },
    formatRegularOpeningHour(openingHour) {
      switch (openingHour.frequency) {
        case "weekly":
          return `${this.weekday(openingHour.weekday)} - ${this.timePeriod(
            openingHour
          )}`;
        case "monthly":
          return `${this.dayOfMonth(
            openingHour.day_of_month
          )} of each month - ${this.timePeriod(openingHour)}`;
        case "fortnightly":
          return `Every other ${this.weekdayFromDate(
            openingHour.starts_at
          )} (${this.fortnightWeek(openingHour.starts_at)}) - ${this.timePeriod(
            openingHour
          )}`;
        case "nth_occurrence_of_month":
          return `${this.dayOfMonth(
            openingHour.occurrence_of_month
          )} ${this.weekday(openingHour.weekday)} of each month`;
      }
    },
    formatHolidayOpeningHour(openingHour) {
      const open = openingHour.is_closed ? "Closed" : "Open";
      const dateRange = `${this.formatDate(
        openingHour.starts_at
      )} to ${this.formatDate(openingHour.ends_at)}`;
      let string = `${open} from ${dateRange}`;

      if (!openingHour.is_closed) {
        string += ` - ${this.timePeriod(openingHour)}`;
      }

      return string;
    },
    timePeriod(openingHour) {
      return `${this.formatTime(openingHour.opens_at)} to ${this.formatTime(
        openingHour.closes_at
      )}`;
    },
    weekday(weekday) {
      return moment(weekday, "E").format("dddd");
    },
    weekdayFromDate(date) {
      return moment(date, moment.HTML5_FMT.DATE).format("dddd");
    },
    dayOfMonth(dayOfMonth) {
      return moment(dayOfMonth, "D").format("Do");
    },
    fortnightWeek(date) {
      const daysInFortnight = 14;
      const thisSunday = moment().day(7);
      const diffInDays = moment(date, moment.HTML5_FMT.DATE).diff(
        thisSunday,
        "days"
      );
      const remainingDays = Math.abs(diffInDays % daysInFortnight);

      return remainingDays > 6 ? "next calendar week" : "this calendar week";
    }
  },
  created() {
    this.fetchAll();
  }
};
</script>
