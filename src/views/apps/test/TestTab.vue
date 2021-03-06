<template>
  <div class="MatcTest">
    <section class="section">
      <div class="container">
        <div class="box is-shadowless">
          <h2 class="title">Users</h2>
          <div class="MatcForm" id="testUserCountCntr">
            <BulletGraph :value="sessionCount" :sections="bulletGraphSection" />
          </div>
        </div>
      </div>
    </section>
    <section class="section" data-dojo-attach-point="sectionDes">
      <div class="container">
        <div class="box is-shadowless">
          <h2 data-nls="testSettingsDescription" class="title">Description</h2>
          <textarea
            class="input"
            v-model="test.description"
            data-gramm_editor="false"
            @change="onDescriptionChange"
            placeholder="Enter here a description that will be shown to your testers."
          ></textarea>
        </div>
      </div>
    </section>

    <section class="section" data-dojo-attach-point="sectionTask">
      <div class="container">
        <div class="box is-shadowless">
          <h2 data-nls="testSettingsTasks" class="title">Tasks</h2>
          <TestSettings :pub="pub" :test="test" :app="app" @change="onTaskChange" />
        </div>
      </div>
    </section>

    <section class="section">
      <div class="container">
        <div class="box is-shadowless">
          <h2 class="title">Screen Recordings</h2>
          <div ref="sessionCntr"></div>
        </div>
      </div>
    </section>

    <section class="section">
      <div class="container">
        <div class="box is-shadowless">
          <h2 class="title">Comments</h2>
          <Comment
            v-if="app"
            :appID="app.id"
            type="overview_test"
            reference
            contentID
            insertPosition="top"
          />
        </div>
      </div>
    </section>
  </div>
</template>
<script>
import Logger from "common/Logger";
import DojoWidget from "dojo/DojoWidget";
import css from "dojo/css";
import lang from "dojo/_base/lang";
import DataFrame from "common/DataFrame";
import BulletGraph from "common/BulletGraph";
import TestSettings from "views/apps/test/TestSettings";
import Table from "common/Table";
import Services from "services/Services";
import Analytics from "dash/Analytics";
// import DomBuilder from 'common/DomBuilder'
import Plan from "page/Plan";
import Util from "core/Util";
import Comment from "page/Comment";

export default {
  name: "Test",
  mixins: [DojoWidget, Plan, Util],
  props: ["app", "test", "annotation", "events"],
  data: function() {
    return {
      sessionCount: 10,
      bulletGraphSection: [
        {
          value: 5,
          color: "#f83a3a",
          label:
            "Jakob Nielsen argues that 5 testers are needed to uncover 85% of all errors in an qualitative usababilty study."
        },
        {
          value: 10,
          color: "#ffb61c",
          label:
            "Hwang & Salvendy suggest the 10+-2 rule as a good sample size."
        },
        {
          value: 20,
          color: "#83b600",
          label:
            "Accoring to Jakob Nielsen 20 testers are required to get statistically valid results."
        }
      ]
    };
  },
  components: {
    BulletGraph: BulletGraph,
    TestSettings: TestSettings,
    Comment: Comment
  },
  computed: {
    pub() {
      return this.$route.meta && this.$route.meta.isPublic;
    },
    urlPrefix() {
      if (this.pub) {
        return "examples";
      }
      return "apps";
    }
  },
  methods: {
    onTaskChange(test) {
      this.$emit("change", lang.clone(test));
    },
    async onDescriptionChange() {
      if (this.pub) {
        this.showSuccess("Saved");
      } else {
        let res = await this.modelService.saveTestSettings(
          this.app.id,
          this.test
        );
        if (res.status === "ok") {
          this.showSuccess("Saved");
          // clone to make sure changes are propagated?
          this.$emit("change", lang.clone(this.test));
        } else {
          this.showError("Could not save test settings");
        }
      }
    },
    showBullet() {
      this.logger.log(-2, "show", "entry >");
      var df = new DataFrame(this.events);
      this.sessionCount = df.unique("session");
      this.logger.log(0, "show", "exit", this.sessionCount);
    },
    showSessions() {
      this.logger.log(-1, "showSessions", "enter " + this.planGetTestCount());

      var maxTestsToShow = this.planGetTestCount();
      let app = this.app;
      var actionEvents = this.getActionEvents(new DataFrame(this.events));
      let events = actionEvents.as_array();
      var list = this._getTestList(events, this.annotation, this.test);
      var urlPrefix = this.urlPrefix;

      /**
       * Set flag to turn off play buttons
       */
      for (var i = 0; i < list.length; i++) {
        var t = list[i];
        if (maxTestsToShow == -1 || i < maxTestsToShow) {
          t.canShow = true;
        } else {
          t.canShow = false;
        }
      }

      var tbl = this.$new(Table);
      tbl.setColumns([
        {
          query: "id",
          label: "#"
        },
         {
          query: "status",
          label: this.getNLS("videoTableStatus")
        },
        {
          query: "user",
          label: this.getNLS("videoTableUser")
        },
        {
          query: "taskPerformance",
          label: "Successful Tasks",
          fct: function(td, row) {
            var names = row.taskNames;
            css.add(td, "MatcDashTableTaskNameCntr tags");
            if (names && names.length > 0) {
              for (var r = 0; r < names.length; r++) {
                var span = document.createElement("span");
                css.add(span, "tag");
                span.innerHTML = names[r];
                td.appendChild(span);
              }
            } else {
              td.innerHTML = "-";
            }
          }
        },
        {
          query: "duration",
          label: this.getNLS("videoTableDuration")
        },
        {
          query: "screens",
          label: this.getNLS("videoTableScreens")
        },
        {
          query: "size",
          label: this.getNLS("videoTableEvents")
        },
        {
          query: "date",
          label: this.getNLS("videoTableDate")
        }
      ]);
      tbl.setActions([
        {
          render: function(node, row) {
            var group = document.createElement("div");
            node.appendChild(group);
            let play = document.createElement("a");
            play.href =
              "#/" +
              urlPrefix +
              "/" +
              app.id +
              "/replay/" +
              row.session +
              ".html";
            css.add(play, "button is-primary");
            play.innerHTML = '<span class="mdi mdi-play"></span>';
            group.appendChild(play);
          }
        }
      ]);

      tbl.placeAt(this.$refs.sessionCntr);
      tbl.setValue(list);
    },

    _getTestList: function(events, annotatation, testSettings) {
      var list = [];

      /**
       * Legacy
       */
      if (!testSettings.tasks) {
        testSettings.tasks = [];
      }

      var df = new DataFrame(events);
      df.sortBy("time");
      var sessionGroup = df.groupBy("session");
      let sessions = sessionGroup.data;

      var annoSession = new DataFrame(annotatation).groupBy("reference");

      var actionDF = this.getActionEvents(new DataFrame(events));

      var analytics = new Analytics();
      var taskCount = testSettings.tasks.length;
      var tasksPerformance = analytics.getMergedTaskPerformance(
        actionDF,
        testSettings.tasks,
        annotatation
      );
      var tasksBySession = tasksPerformance.count("session");

      var sessionTaskNames = analytics.getSuccessTaskNames(
        tasksPerformance,
        testSettings.tasks
      );

      var id = 1;
      for (let sessionID in sessions) {
        var session = sessions[sessionID];
        var date = this.formatDate(session.min("time"));

        var anno = annoSession.get(sessionID);
        var status = '<span class="MatchDashStatusSuccess">Valid</span>';
        var isValid = true;
        if (anno) {
          isValid = anno.get(0).get("isValid");
          if (!isValid) {
            status = '<span class="MatchDashStatusFailure">Failure</span>';
          }
        }


        /** Since 2.4 we show also the user */
        let user = session.data && session.data.length > 0 ? session.data[0].user : '-'
        if (user && user.name) {
          user = user.name
        }

        var taskSuccess = tasksBySession.get(sessionID);
        if (!taskSuccess) {
          taskSuccess = 0;
        }

        var item = {
          session: sessionID,
          user: user,
          taskPerformance: taskSuccess + " / " + taskCount,
          taskNames: sessionTaskNames[sessionID],
          duration: Math.ceil(
            (session.max("time") - session.min("time")) / 1000
          ),
          date: date,
          start: session.min("time"),
          size: session.size(),
          status: status,
          isValid: isValid,
          id: id,
          screens: session.unique("screen")
        };

        list.push(item);
        id++;
      }

      list.sort(function(a, b) {
        return a.id - b.id;
      });

      return list;
    }
  },
  watch: {
    test(v) {
      // for some reason not called
      this.logger.info("watch", "test >", v);
      this.test = v;
    }
  },
  async mounted() {
    this.logger = new Logger("Test");
    this.modelService = Services.getModelService(this.$route);
    this.showBullet();
    this.showSessions();
    this.logger.info("mounted", "exit");
  }
};
</script>

