<template>
  <q-page padding>
    <div class="q-pb-md q-gutter-sm">
      <q-breadcrumbs>
        <q-breadcrumbs-el label="Home" />
      </q-breadcrumbs>
      <div class="column items-end">
        <div class="col">
          <q-btn-dropdown color="primary" label="Escolha o mês">
            <q-list>
              <q-item
                v-for="item in months"
                :key="item"
                clickable
                v-close-popup
                @click="onAlterMonth(item.index)"
              >
                <q-item-section>
                  <q-item-label>{{ item.mes }}</q-item-label>
                </q-item-section>
              </q-item>
            </q-list>
          </q-btn-dropdown>
        </div>
      </div>
      <div class="row">
        <div class="col">
          <q-card style="width: 100%" flat bordered>
            <q-card-section horizontal style="width: 100%">
              <q-card-section class="col-6">
                <h5 class="text-overline q-ma-none q-pb-sm">
                  GALERA NA ATIVA:
                </h5>
                <div v-if="ticketsInDevelop">
                  <div
                    class="row text-grey-8 q-gutter-xs"
                    v-for="ticket in ticketsInDevelop"
                    :key="ticket?.id"
                  >
                    <q-chip size="sm" v-if="ticket?.collaborator">
                      <q-avatar v-if="ticket?.collaborator?.image">
                        <img
                          :src="`https://cajueiroapi.cajutec.com.br/storage/images/${ticket.collaborator.image.uri}`"
                        />
                      </q-avatar>
                      {{ ticket.collaborator?.first_name }}
                    </q-chip>
                    <q-badge
                      rounded
                      flat
                      class="text-caption text-weight-regular"
                      :style="`background:${
                        status[ticket?.status]?.hex
                      }; font-size: 10px;`"
                      :label="`#${ticket?.code}`"
                    />
                  </div>
                </div>
                <div v-else class="items-center text-center q-pa-sm">
                  <q-spinner-cube color="blue" size="3.5em" />
                </div>
              </q-card-section>

              <q-separator vertical />

              <q-card-section class="col">
                <h5 class="text-overline q-ma-none q-pb-sm">PLANTÃO:</h5>

                <DutiesComponent></DutiesComponent>
              </q-card-section>
            </q-card-section>
          </q-card>
        </div>
      </div>
      <apex-bar class="column q-mt-lg" :month="monthSelect"></apex-bar>
      <div class="row q-gutter-md justify-center items-center q-mt-lg">
        <q-card
          dark
          bordered
          class="bg-blue-9 my-card q-col-12 q-col-md-6 q-mb-md"
        >
          <q-card-section>
            <div class="text-h6">Média de protocolos finalizados</div>
            <div
              class="text-subtitle2"
              v-if="!isNaN(getResults().averangCount)"
            >
              <h1 class="text-h1 q-pa-none q-ma-none">
                {{ getResults().averangCount }}
              </h1>
            </div>
          </q-card-section>

          <q-separator dark inset />
          <q-card-section v-if="!isNaN(getResults().averangCount)">
            <ul>
              <li v-for="item in getResults().upCount" :key="item">
                {{ item.user }} |
                {{ item.count }}
              </li>
            </ul>
          </q-card-section>
          <q-card-section
            class="items-center text-center q-ma-lg"
            v-if="isNaN(getResults().averangCount)"
          >
            <q-spinner-cube color="while" size="5.5em" />
          </q-card-section>
        </q-card>
        <q-card
          dark
          bordered
          class="bg-blue-9 my-card q-col-12 q-col-md-6 q-mb-md"
        >
          <q-card-section>
            <div class="text-h6">Média de pontos de protocolos</div>
            <div
              class="text-subtitle2"
              v-if="!isNaN(getResults().averangCount)"
            >
              <h1 class="text-h1 q-pa-none q-ma-none">
                {{ getResults().averangPoints }}
              </h1>
            </div>
          </q-card-section>

          <q-separator dark inset />
          <q-card-section v-if="!isNaN(getResults().averangCount)">
            <ul>
              <li v-for="item in getResults().upPoints" :key="item">
                {{ item.user }} |
                {{ item.points }}
              </li>
            </ul>
          </q-card-section>
          <q-card-section
            class="items-center text-center q-ma-lg"
            v-if="isNaN(getResults().averangCount)"
          >
            <q-spinner-cube color="while" size="5.5em" />
          </q-card-section>
        </q-card>
      </div>
    </div>
  </q-page>
</template>

<script>
import { defineComponent, onMounted, ref, reactive, provide, watch } from 'vue';
import ApexBar from 'src/components/charts/apexBar.vue';
import DutiesComponent from 'src/components/dashboard/DutiesComponent.vue';
import ticketsService from 'src/services/tickets';
import _ from 'lodash';
import status from 'src/support/tickets/status';
import priority from 'src/support/tickets/priority';

export default defineComponent({
  name: 'IndexPage',
  components: { ApexBar, DutiesComponent },
  setup() {
    onMounted(async () => {
      await getTickets();
      await getTicketsInDevelop();
    });

    const options = ref();
    const tickets = ref();
    const ticketsInDevelop = ref();
    const dataCollaborators = ref();
    const loading = reactive({
      results: true,
    });
    const months = [
      { index: 1, mes: 'JAN' },
      { index: 2, mes: 'FEV' },
      { index: 3, mes: 'MAR' },
      { index: 4, mes: 'ABR' },
      { index: 5, mes: 'MAI' },
      { index: 6, mes: 'JUN' },
      { index: 7, mes: 'JUL' },
      { index: 8, mes: 'AGO' },
      { index: 9, mes: 'SET' },
      { index: 10, mes: 'OUT' },
      { index: 11, mes: 'NOV' },
      { index: 12, mes: 'DEZ' },
    ];
    const { myTickets, ticketsGraphUsers } = ticketsService();
    const monthSelect = ref(Number(0));

    const onAlterMonth = (month) => {
      monthSelect.value = month;
      getTickets();
    };

    const averang = (num = null, qntItem = null) => {
      return parseFloat(num / qntItem);
    };

    const recoverCollaborators = async () => {
      dataCollaborators.value = await _.chain(tickets.value)
        .groupBy('collaborator_id')
        .map((value) => recoverUser(value))
        .filter((value) => value.user !== undefined)
        .value();
    };

    const dateActual = () => {
      const date = new Date();
      const month = date.getMonth();
      return parseInt(month) + 1;
    };

    const recoverUser = (collaborator) => {
      const payload = reactive({
        user: null,
        count: 0,
        points: 0,
        priority: 0,
      });

      _.forEach(collaborator, function (value) {
        payload.user = value?.collaborator?.first_name;
        payload.count = collaborator?.length;
        payload.points +=
          parseFloat(value?.impact?.points) +
          (value.priority === 'yes' ? 1 : 0);
        payload.priority += value.priority === 'yes' ? 1 : 0;
      });

      return payload;
    };

    const getTickets = async () => {
      try {
        // tickets.value = await list();
        tickets.value = await ticketsGraphUsers('?filter[status]=done', {
          month: monthSelect.value !== '' ? monthSelect.value : dateActual(),
        });
        await recoverCollaborators();
        const data = ref({
          xaxis: {
            categories: [...dataCollaborators.value.map((v) => v.user)],
          },
          finals: {
            sum: _.sum([...dataCollaborators.value.map((v) => v.count)]),
            averang: _.meanBy([...dataCollaborators.value.map((v) => v.count)]),
            elem: [...dataCollaborators.value.map((v) => v.count)],
          },
          points: {
            sum: _.sum([...dataCollaborators.value.map((v) => v.points)]),
            averang: _.meanBy([
              ...dataCollaborators.value.map((v) => v.points),
            ]),
            elem: [...dataCollaborators.value.map((v) => v.points)],
          },
        });

        options.value = await { ...data.value };
      } catch (error) {
        console.log(error);
      }
    };

    const getTicketsInDevelop = async () => {
      try {
        const data = await myTickets(
          '?include=collaborator.image,impact,user.collaborator,client.corporate.image,&filter[collaborator_id]=&filter[status]=development'
        );
        ticketsInDevelop.value = data;
      } catch (error) {
        console.log(error);
      }
    };

    const getResults = () => {
      const countSum = ref(
        averang(
          _.sumBy(dataCollaborators.value, function (value) {
            return value.count;
          }),
          _.size(dataCollaborators.value)
        )
      );

      const pointsSum = ref(
        averang(
          _.sumBy(dataCollaborators.value, function (value) {
            return value.points + (value.priority === 'yes' ? 1 : 0);
          }),
          _.size(dataCollaborators.value)
        )
      );

      return {
        averangCount: countSum.value.toFixed(2),
        averangPoints: pointsSum.value.toFixed(2),
        upCount: _.filter(dataCollaborators.value, function (v) {
          return v.count >= countSum.value;
        }),
        upPoints: _.filter(dataCollaborators.value, function (v) {
          return v.points >= pointsSum.value;
        }),
      };
    };

    return {
      options,
      dataCollaborators,
      getResults,
      monthSelect,
      averang,
      ticketsInDevelop,
      status,
      priority,
      loading,
      months,
      onAlterMonth,
    };
  },
});
</script>
