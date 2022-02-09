<template>
  <div>
    <div
      class="has-background-grey-dark has-text-centered p-2 has-text-white has-text-weight-semibold is-flex is-justify-content-center is-align-items-center"
    >
      <img
        src="../assets/icons/icon_32.png"
        class="mr-2"
      > Domain Expiration
      Checker
    </div>
    <section class="p-2">
      <div
        class="control"
        :class="isLoading ? 'is-loading' : ''"
      >
        <input
          class="input"
          type="text"
          :value="domain"
          placeholder="Fetching details..."
          disabled
        >
        <div
          v-if="isLoading"
          class="mt-2 p-2 has-text-centered w-100"
        >
          Fetching details...
        </div>
        <div
          v-if="error"
          class="block mt-2"
        >
          <article class="message is-danger is-small">
            <div class="message-body">
              {{ message }}
            </div>
          </article>
        </div>
        <div
          v-else
          class="block mt-2"
        >
          <div
            v-for="(event, index) in filteredEvents"
            :key="index"
            class="p-2 w-100 is-size-7 is-flex is-justify-content-space-between"
          >
            <span
              class="is-capitalized has-text-weight-semibold"
            >{{ event.eventAction }}
            </span>
            <span>{{ event.eventDate }} </span>
          </div>
          <a
            v-if="calenderLink"
            :href="calenderLink"
            class="button is-small is-fullwidth is-primary mt-2"
            target="_blank"
          >Add to calender</a>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
export default {
  data() {
    return {
      domain: '',
      isLoading: true,
      error: false,
      link: '',
      events: [],
      url: 'https://rdap.verisign.com/com/v1/domain/',
      actions: ['expiration', 'registration', 'last updated'],
      message: 'Something went wrong',
      calenderLink: '',
    }
  },
  computed: {
    filteredEvents () {
      return this.events.filter((event) => {
        if (event.eventAction === 'expiration') {
          const expiry = event.eventDate.replaceAll('-', '')
          this.calenderLink = `https://www.google.com/calendar/render?action=TEMPLATE&text=Renew+domain+${this.domain}&dates=${expiry}/${expiry}&details=&location=&sf=true&output=xml`
        }
        return this.actions.includes(event.eventAction)
      })
    },
  },
  mounted() {
    chrome.tabs.query({ currentWindow: true, active: true }, (tabs) => {
      const url = new URL(tabs[0].url)
      const hostname = url.hostname.split('.')
      const [domainName, tld] = hostname.slice(-2)
      this.domain = `${domainName}.${tld}`

      this.$nextTick(() => {
        if (tld !== 'com') {
          chrome.storage.local.get(['dns'], (result) => {
            const dns = result.dns.find((d) => d.tld === tld)

            if (dns) {
              this.url = dns.https
              this.fetchDetails()
            } else {
              this.error = true
              this.message = `'.${tld}' top level domain is not supported.`
              this.isLoading = false
            }
          })
        } else {
          this.fetchDetails()
        }
      })
    })
  },
  methods: {
    fetchDetails() {
      console.log(this.url)
      fetch(`${this.url}${this.domain.toUpperCase()}`)
        .then((response) => {
          if (!response.ok) {
            this.isLoading = false
            this.error = true
            return
          }
          return response.json()
        })
        .then((data) => {
          const links = data.links || []
          console.log(this.links)
          if (!links.length) {
            this.isLoading = false
            this.error = true
            return
          }
          if (links.length === 1) {
            this.link = links[0]
          }

          if (this.link.events) {
            this.events = this.link.events

            this.isLoading = false
            return
          }
          this.link = links.find((lnk) => lnk.rel === 'related')

          if (this.link) {
            this.domainLookup(this.link.value)
          }
        })
        .catch(() => {
          this.isLoading = false
          this.error = true
        })
    },
    domainLookup(url) {
      fetch(url)
        .then((response) => {
          if (!response.ok) {
            this.isLoading = false
            this.error = true
            return
          }
          return response.json()
        })
        .then((data) => {
          this.events = data.events
          this.isLoading = false
        })
        .catch(() => {
          this.isLoading = false
          this.error = true
        })
    },
  },
}
</script>

<style lang="sass"></style>
