<template>
  <v-app>
    <v-toolbar app>
      <v-toolbar-title class="headline text-uppercase">
        <span>Airdrop</span>
      </v-toolbar-title>
      <v-spacer></v-spacer>
    </v-toolbar>

    <v-content>
      <v-card>
        <v-container
                fluid
                grid-list-lg
        >
          <v-layout row wrap>
            <v-flex xs12>
              <v-card>
                <v-card-title primary-title>
                    <h2>
                      Signed wallet
                    </h2>
                    <v-form
                        ref="form"
                        v-model="valid"
                        lazy-validation
                        style="width: 100%"
                    >
                      <v-radio-group v-model="env">
                        <v-radio
                          label="Test"
                          value="test"
                        ></v-radio>
                        <v-radio
                          label="Live"
                          value="prod"
                        ></v-radio>
                      </v-radio-group>
                      <v-text-field
                              v-model="accessToken"
                              label="Access Token"
                              required
                      ></v-text-field>
                      <v-text-field
                              v-model="otp"
                              label="OTP"
                              required
                      ></v-text-field>
                      <v-text-field
                              v-model="symbol"
                              label="Symbol"
                              required
                      ></v-text-field>
                      <v-text-field
                              v-model="walletId"
                              label="Wallet Id"
                              required
                      ></v-text-field>

                      <v-text-field
                              v-model="walletPassphrase "
                              label="Wallet Passphrase"
                              required
                      ></v-text-field>
                    </v-form>
                </v-card-title>
              </v-card>
            </v-flex>

            <v-flex xs12>
              <v-card>
                <v-layout>
                  <v-flex xs12>
                    <v-card-title primary-title>
                      <div style="width: 100%">
                        <v-toolbar flat color="white">
                          <v-toolbar-title>CSV for Airdrop</v-toolbar-title>
                          <v-spacer></v-spacer>
                          <input ref="csv" type="file" name="csv" />
                          <v-btn dark @click.prevent="importCSV">
                            Import CSV
                          </v-btn>
                        </v-toolbar>
                        <v-data-table
                                :headers="headers"
                                :items="sendList"
                                item-key="name"
                                :pagination.sync="pagination"
                        >
                          <template v-slot:items="props">
                            <tr>
                              <td>{{ props.item.address }}</td>
                              <td>{{ props.item.memo }}</td>
                              <td class="text-xs-right">{{ props.item.amount }}</td>
                            </tr>
                          </template>
                        </v-data-table>
                      </div>
                    </v-card-title>
                    <v-card-actions>
                      <v-btn flat color="orange" @click="sendAll()">Send All</v-btn>
                    </v-card-actions>
                  </v-flex>
                </v-layout>
              </v-card>
            </v-flex>
            <v-flex xs12>
              <v-card>
                <v-card-title><h4>Result</h4></v-card-title>
                <v-divider></v-divider>
                <v-list dense>
                  <v-list-tile v-for="item in resultList">
                    <v-list-tile-content>[{{item.status}}]  {{item.address}}</v-list-tile-content>
                    <v-list-tile-content class="align-end">{{item.amount}}</v-list-tile-content>
                  </v-list-tile>
                </v-list>
              </v-card>
            </v-flex>
          </v-layout>
        </v-container>
      </v-card>
    </v-content>
  </v-app>
</template>

<script>
import _ from 'lodash';
import Papa from 'papaparse';
import rp from 'request-promise';


export default {
  name: 'App',
  components: {
  },
  data () {
    return {
      valid: true,
      env: 'test',
      accessToken: 'v2x275c2a2b5969640ae24e698192bf4fedc9a03554e161cbc6cea1812caec3a0e1',
      otp: '000000',
      walletId: '5cc5a5d4d5a49b7f03e1ef7833ab0a79',
      walletPassphrase: 'dkahsem!234',
      symbol: 'terc',
      cvs: null,
      headers: [
        {
          text: 'Address',
          align: 'left',
          sortable: false,
          value: 'address'
        },
        { text: 'Memo', value: 'memo', sortable: false },
        { text: 'Amount', value: 'amount', sortable: false },
      ],
      pagination: {
        rowsPerPage: -1
      },
      sendList: [],
      resultList: [],
    }
  },
  methods: {
    validate () {
      if (this.$refs.form.validate()) {
        this.snackbar = true
      }
    },
    async sendAll() {
      console.log(this);
      if(this.sendList.length === 0){
        alert('No data.');
        return;
      }
      if(this.env == ''){
        alert('Test or Live is required.');
        return;
      }
      if(this.walletId == '' ){
        alert('Wallet ID is required.');
        return;
      }
      if(this.otp == '' ){
        alert('OTP is required.');
        return;
      }
      if(this.symbol == '' ){
        alert('Symbol is required.');
        return;
      }
      if(this.accessToken == '' ){
        alert('Access Token is required.');
        return;
      }

      this.resultList = [];
      const recipients = [];
      this.sendList.forEach(data => {
        recipients.push({
          address: data.address,
          amount: data.amount
        });
      })

      const result = await rp({
        uri: `http://localhost:8081/api/sends`,
        method: 'POST',
        body: {
          recipients,
          otp: this.otp,
          symbol: this.symbol,
          walletId: this.walletId,
          walletPassphrase: this.walletPassphrase,
          accessToken: this.accessToken,
        },
        json: true
      })
      this.resultList = result;
    },
    importCSV() {
      const _this = this;
      _this.sendList = [];

      this.readFile((output) => {
        _this.csv = _.get(Papa.parse(output), "data");
        if(_this.csv){
          _this.csv.forEach(data => {
            _this.sendList.push({
              address: data[0],
              memo: data[1],
              amount: data[2],
            })
          })
        }
      });
    },
    readFile(callback) {
      let file = this.$refs.csv.files[0];

      if (file) {
        let reader = new FileReader();
        reader.readAsText(file, "UTF-8");
        reader.onload = function (evt) {
          callback(evt.target.result);
        };
        reader.onerror = function () {
        };
      }
    }
  }
}
</script>
