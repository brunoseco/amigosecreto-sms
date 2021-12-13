<template>
    <div id="app">

        <div class="container p-5">
            <div class="row">
                <div class="col">
                    <div class="card">
                        <div class="card-body">
                            <h5 class="card-title">Insira o número e nome dos participantes</h5>
                            <h6 class="card-subtitle mb-2 text-muted">Insira um participante por linha, primeiro nome e depois telefone, separado por ;</h6>
                            <textarea placeholder="Bruno;5515999999999"
                                      v-model="textoParticipantes"
                                      class="form-control"
                                      @blur="calculaParticipantes"
                                      rows="5" />
                        </div>
                    </div>

                    <div class="card mt-3">
                        <div class="card-body">
                            <h5 class="card-title">Chave token SMS</h5>
                            <h6 class="card-subtitle mb-2 text-muted">Chave da api do Zenvia</h6>
                            <input type="text" class="form-control" v-model="textoToken" />
                        </div>
                    </div>

                    <div class="card mt-3">
                        <div class="card-body">
                            <h5 class="card-title">Texto da mensagem</h5>
                            <h6 class="card-subtitle mb-2 text-muted">Insira um texto complementar</h6>
                            <textarea placeholder="Amigo chocolate"
                                      v-model="textoMensagem"
                                      class="form-control"
                                      rows="4" />
                        </div>
                    </div>
                </div>
                <div class="col">
                    <div class="card">
                        <div class="card-body">
                            <h5 class="card-title">Lista dos participantes formatada</h5>
                            <ol class="list-group list-group-numbered" v-if="participantes && participantes.length > 0">
                                <li class="list-group-item d-flex justify-content-between align-items-start"
                                    v-for="participante in participantes" v-bind:key="participante.numero">
                                    <div class="ms-2 me-auto">
                                        <div class="fw-bold">{{ participante.nome }}</div>
                                        {{ participante.numero }}
                                    </div>
                                </li>
                            </ol>
                            <h6 class="card-subtitle mb-2 text-muted" v-else>Insira os participantes no passo anterior</h6>
                        </div>
                    </div>

                    <div class="card mt-3">
                        <div class="card-body">
                            <h5 class="card-title">Prévia da mensagem</h5>
                            <h6 class="card-subtitle mb-2 text-muted">
                                Olá <strong>NOME 1</strong>, seu amigo(a) secreto(a) é <strong>NOME 2</strong>. {{ textoMensagem }}
                            </h6>
                        </div>
                    </div>

                    <div class="card mt-3">
                        <div class="card-body">
                            <h5 class="card-title">Tudo pronto?</h5>
                            <h6 class="card-subtitle mb-2 text-muted">Confira todas as informações acima</h6>
                            <button type="button" class="btn w-100 btn-success" @click="processar()">Processar e enviar</button>
                        </div>
                    </div>
                </div>
                <div class="col">

                    <div class="card">
                        <div class="card-body">
                            <h5 class="card-title">Resultado do envio</h5>
                            <ol class="list-group list-group-numbered" v-if="jsonEnvio && jsonEnvio.length > 0">
                                <li class="list-group-item d-flex justify-content-between align-items-start"
                                    v-for="item in jsonEnvio" v-bind:key="item.numero">
                                    <div class="ms-2 me-auto" v-if="item.status != 'Pendente'">
                                        <div class="fw-bold">{{ item.nome }}</div>
                                        {{ item.status }}
                                    </div>
                                </li>
                            </ol>
                            <h6 class="card-subtitle mb-2 text-muted" v-else>Aguardando processamento</h6>
                        </div>
                    </div>
                </div>
            </div>
        </div>

    </div>
</template>

<script>

    const { post } = require('request-promise');

    export default {
        name: 'App',
        data() {
            return {
                textoParticipantes: null,
                textoToken: null,
                textoMensagem: null,
                participantes: [],
                participantesAtribuidos: [],
                jsonEnvio: null
            }
        },
        methods: {
            calculaParticipantes() {
                this.participantes = [];
                let participantes = this.textoParticipantes.split("\n");
                for (var i = 0; i < participantes.length; i++) {
                    const items = participantes[i].split(";");
                    if (items && items.length && items.length > 1) {
                        const nome = items[0];
                        const numero = items[1];
                        this.participantes.push({
                            nome: nome ? nome.trim().toUpperCase() : "",
                            numero: nome ? numero.match(/\d+/g).join("") : ""
                        });
                    }
                }
            },

            embaralhaParticipantes() {
                for (let i = this.participantes.length - 1; i > 0; i--) {
                    const j = Math.floor(Math.random() * (i + 1));
                    [this.participantes[i], this.participantes[j]] = [this.participantes[j], this.participantes[i]];
                }
            },

            atribuiParticipantes() {
                this.embaralhaParticipantes();
                this.participantesAtribuidos = [];
                for (let i = 0; i < this.participantes.length; i++) {
                    const p1 = this.participantes[i];
                    const p2 = this.participantes[i + 1] || this.participantes[0];
                    this.participantesAtribuidos.push({ p1, p2 });
                }
            },

            configuraEnvio() {
                this.jsonEnvio = this.participantesAtribuidos.map((assign) => {
                    return {
                        nome: assign.p1.nome,
                        numero: assign.p1.numero,
                        status: "Pendente",
                        mensagem: `Olá ${assign.p1.nome}, seu amigo(a) secreto(a) é ${assign.p2.nome}. ${this.textoMensagem}`,
                    }
                })
            },

            enviarApi() {

                if (!this.textoToken)
                    return;

                this.jsonEnvio.forEach(item => {

                    post({
                        uri: 'https://api.zenvia.com/v2/channels/sms/messages',
                        headers: {
                            'X-API-TOKEN': this.textoToken
                        },
                        body: {
                            from: '5515998027734',
                            to: item.numero,
                            contents: [{
                                type: 'text',
                                text: item.mensagem
                            }]
                        },
                        json: true
                    })
                        .then(() => {
                            item.status = "Sucesso";
                        })
                        .catch((error) => {
                            item.status = "Erro, olhe a aba de console F12";
                            console.log('Error:', error);
                        });

                });
            },

            processar() {
                this.atribuiParticipantes();
                this.configuraEnvio();
                this.enviarApi();
            }
        }
    }
</script>

<style>
</style>
