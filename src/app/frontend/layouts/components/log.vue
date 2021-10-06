<template>
  <div class="main-log" v-if="tabCounter > 0">
    <section class="content margin-right">
      <div class="card card-secondary card-outline">
        <div class= "card-body p-2 ">
          <b-tabs card grow active-nav-item-class="font-weight-bold" v-model="active_tab" ref="tabs">
            <!-- Render Tabs, supply a unique `key` to each tab -->
            <b-tab v-for="val in tabs" :key="val.index" class="text-truncate">
              <template #title>
                {{val.podName}}
                <button type="button" class="btn btn-tool" @click="closeTab(val.containerName, val.podName)" ><i class="fas fa-times"></i></button>
              </template>
              <div class="overflow-auto " style="height:15rem;">
                <div v-if="val.data === '' ">
                  {{val.containerName}} : No Log....
                </div>
                <div v-else class="row mb-1 mx-1" v-for="d in val.data">
                  <div class="col-sm-12">
                    {{val.containerName}} : {{d}}
                  </div>
                </div>
              </div>
            </b-tab>
          </b-tabs>
        </div>
      </div>
    </section>
  </div>
</template>
<script>
export default {
  data() {
    return {
      tabs: [],
      active_tab: 0,
      tabCounter: 0
    }
  },
  created() {
    this.$nuxt.$on('log-data-created', (metadata, containerName) => {
      if (metadata.name) {
        if (this.tabCounter < 10) {
          this.contentPadding('25rem');

          let flag = true;
          this.tabs.forEach(tabs => {
            if (tabs.containerName === containerName && tabs.podName === metadata.name) {
              flag = false;
              this.active_tab = tabs.index;
            }
          })
          if (flag) {
            this.tabs.push({
              index: this.tabCounter,
              containerName: containerName,
              podName: metadata.name,
              response: {},
              data: ''
            })
            this.tabCounter++;
            setTimeout(() => {
              this.active_tab = this.tabs[this.tabs.length - 1].index;
            })

            let baseUrl = `${this.$config.nodeEnv}` === "development" ? `${location.protocol}//${location.hostname}:${this.$config.backendPort}` : `${location.host}`;
            fetch(baseUrl + this.getApiUrl("", "pods", metadata.namespace, metadata.name, "follow=1&container=" + containerName + "&sinceSeconds=600", true)).then(response => {
              const reader = response.body.getReader();
              const Vue = this;
              let buffer = '';
              this.tabs[this.tabs.length - 1].response = {containerName: containerName, podName: metadata.name, read: reader};

              return reader.read().then(function process(result) {
                if (result.done) {
                  return;
                }
                buffer = new TextDecoder().decode(result.value, {stream: true});

                Vue.tabs.forEach(tabs => {
                  if (tabs.containerName === containerName && tabs.podName === metadata.name) {
                    const logData = buffer.replace(/\[(?:\d{1}|\d{2})m/gi, '').split("\n");
                    tabs.data = tabs.data.length === 0 ? logData.filter(item => item.length > 0) : tabs.data.concat(logData).filter(item => item.length > 0);
                    setTimeout(() => {
                      const logDiv = Vue.$refs.tabs.$refs.content;
                      for (let i = 0; i < logDiv.children.length; i++) {
                        logDiv.children[i].children[0].scrollTop = logDiv.children[i].children[0].scrollHeight;
                      }
                    });
                  }
                })
                return reader.read().then(process);
              })
            })
          }
        }
      }
    });
  },
  methods: {
    closeTab(containerName, podName) {
      let deleteIdx = 10;
      this.tabs.forEach((item, index) => {
        if (item.containerName === containerName && item.podName === podName) {
          if (item.response.hasOwnProperty("read")) {
            item.response.read.cancel();
          }
          this.tabs.splice(index, 1);
        }
      })
      if(this.tabs.length === 0){
        this.contentPadding('2rem');
      }
    }
  }
}
</script>