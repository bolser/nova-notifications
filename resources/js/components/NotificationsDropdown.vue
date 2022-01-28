<template>
  <dropdown>
    <dropdown-trigger>
      <i class="pi pi-notification w-25 h-30"></i>
    </dropdown-trigger>

        <dropdown-menu slot="menu" width="600" direction="rtl">
            <loading-view :loading="isLoading">
                <div class="flex justify-between bg-40 text-90 p-4">
                    <h3>{{ __('Notifications') }}</h3>

                    <button v-if="count !== 0" class="btn" @click="markAllAsRead()">
                        {{ __('mark all as read') }}
                    </button>
                </div>
                <p v-if="count === 0" class="block p-3">
                    {{ __('No new notifications') }}
                </p>
                <scroll-wrap v-else height="350">
                    <slot>
                        <notification-item
                            :ref="'notification-' + notification.id"
                            v-for="notification in notifications"
                            :key="notification.id"
                            :notification="notification"
                        >
                        </notification-item>
                    </slot>
                </scroll-wrap>
            </loading-view>
        </dropdown-menu>
    </dropdown>
</template>

<script>
    export default {
        data() {
            return {
                count: 0,
                notifications: [],
                isLoading: true,
                sound: Nova.config.default_sound,
            }
        },
        mounted() {
            const self = this

            self.loadNotifications(function (response) {
                Echo.private(Nova.config.user_model_namespace + '.' + response.data.user_id)
                    .notification(self.notificationReceived)
            })

            Nova.$on('notification-read', function (e) {
                self.notifications[e.notification.id] = e.notification
                self.count--;
            })
        },
        methods: {
            loadNotifications: function (callback) {
                axios
                    .get('/nova-vendor/nova-notifications/unread')
                    .then(response => {
                        this.isLoading = false
                        this.count = response.data.count
                        this.notifications = response.data.notifications

                        if (callback) {
                            callback(response)
                        }
                    })
            },
            playSound: function (sound = this.sound) {
                if(Nova.config.play_sound){
                    new Audio(sound).play();
                }
            },
            notificationReceived: function (notification) {
                const self = this

                self.loadNotifications()

                let level = 'info'
                const levels = ['success', 'info', 'error']

                if (levels.indexOf(notification.level) !== -1) {
                    level = notification.level
                }

                const markAsRead = {
                    text: self.__('Mark as Read'),
                    onClick: (e, toast) => {
                        self.$refs['notification-' + notification.id][0].markAsRead()
                        toast.goAway(0);
                    }
                }

                const cancel = {
                    text: self.__('Cancel'),
                    onClick: (e, toast) => {
                        toast.goAway(0);
                    }
                }

                let actions = []

                if (notification.show_mark_as_read) {
                    actions.push(markAsRead)
                }

                if (notification.show_cancel) {
                    actions.push(cancel)
                }

                self.$toasted.show(notification.title, {
                    type: level,
                    keepOnHover: true,
                    icon: notification.icon || null,
                    iconPack: 'custom-class',
                    action: actions,
                })

                this.playSound(notification.sound)
            },
            markAllAsRead: function () {
                axios
                    .patch('/nova-vendor/nova-notifications')
                    .then(() => {
                        this.notifications = []
                        this.count = 0
                    })
            }
        }
    }
</script>

<style scoped>
    #max-height {
        max-height: 30rem;
    }

    .bg-light-gray {
        background-color: #EEEEEE;
    }

    .bg-light-gray:hover {
        background-color: #fefefe;
    }
</style>
