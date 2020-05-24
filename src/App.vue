<template>
	<v-app>
		<v-navigation-drawer app v-model="drawer">
			<v-list-item>
				<v-list-item-content>
					<v-list-item-title class="title">
						Graf
					</v-list-item-title>
					<v-list-item-subtitle>
						add item
					</v-list-item-subtitle>
				</v-list-item-content>
			</v-list-item>

			<v-divider></v-divider>

			<template>
				<v-layout row wrap>
					<v-flex xs12>
						<v-card v-for="(item, index) in list" :key="index">
							<v-container fluid grid-list-lg>
								<v-layout row>
									<v-flex xs4 ml-3>
										<v-card-title primary-title>
											{{ item.tree.id }}
										</v-card-title>
									</v-flex>
									<v-flex xs6 row>
										<v-flex xs11 class="pa-1">
											<!-- {{ item.tree.child }} -->
											<v-select single-line chipS :items="NodesToArrayIdNot(item.tree.id)" v-model="item.tree.child" label="nodes" multiple @input="select(item.tree.child, item.tree.id)"></v-select>
										</v-flex>
										<v-flex xs1 class="pa-1">
											<v-icon @click="del(item.tree.id)">mdi-delete</v-icon>
										</v-flex>
									</v-flex>
								</v-layout>
							</v-container>
						</v-card>
					</v-flex>
				</v-layout>
			</template>
			<v-divider></v-divider>
			<!-- 			<v-btn block color="primary" dark @click="add">Add</v-btn>
			<v-btn block color="done" dark @click="save">Save</v-btn>
			<v-divider></v-divider> -->
			<v-text-field class="mx-2" type="number" append-icon="mdi-plus red--text" prepend-icon="mdi-minus green--text " label="Lenght" v-model.number="params.l0" @click:append="params.l0 += 10" @click:prepend="params.l0 -= 10"> </v-text-field>
			<v-text-field class="mx-2" type="number" append-icon="mdi-plus red--text" prepend-icon="mdi-minus green--text " label="Gamma" v-model.number="params.gamma" @click:append="params.gamma += 0.1" @click:prepend="params.gamma -= 0.1"> </v-text-field>
			<v-text-field class="mx-2" type="number" append-icon="mdi-plus red--text" prepend-icon="mdi-minus green--text " label="K" v-model.number="params.k" @click:append="params.k += 0.1" @click:prepend="params.k -= 0.1"> </v-text-field>
			<v-text-field class="mx-2" type="number" append-icon="mdi-plus red--text" prepend-icon="mdi-minus green--text " label="q2" v-model.number="params.q2" @click:append="params.q2 += 10" @click:prepend="params.q2 -= 10"> </v-text-field>
		</v-navigation-drawer>
		<v-app-bar app color="primary" dark>
			<!-- <v-icon class="mr-4" v-if="!drawer" @click="drawer = !drawer">mdi-chevron-right</v-icon> -->
			<v-app-bar-nav-icon class="mr-4" v-if="!drawer" @click="drawer = !drawer"></v-app-bar-nav-icon>
			<v-icon class="mr-4" v-else @click="drawer = !drawer">mdi-chevron-left</v-icon>
			<v-spacer></v-spacer>
			<v-tooltip bottom>
				<template v-slot:activator="{ on }">
					<v-icon @click="test" v-on="on">mdi-alpha-t-box-outline</v-icon>
				</template>
				<span>Testing graf</span>
			</v-tooltip>
			<v-spacer></v-spacer>
			<v-icon class="mx-4" @click="reload = !reload">mdi-reload</v-icon>
			<v-icon class="mx-4" @click="add">mdi-plus</v-icon>
			<v-icon class="mx-4" @click="save">mdi-content-save</v-icon>
		</v-app-bar>

		<v-content>
			<Main :tree="tree" :params="params" :reload="reload" />
		</v-content>
	</v-app>
</template>

<script>
	import Main from './components/Main';
	export default {
		name: 'App',
		components: {
			Main,
		},
		created() {},
		mounted() {
			/** @type {Array} */
			let nodes = eval(localStorage.nodes);
			nodes.forEach((e, i) => {
				let n = new Node(
					i,
					e
						.map((l, ii) => {
							if (l) {
								return ii;
							}
						})
						.filter((e) => e != undefined),
				);
				/* this.tree.push(n); */
				this.list.push({ tree: n });
			});
		},
		data: () => ({
			reload: false,
			params: {
				gamma: 2,
				l0: 200,
				k: 1,
				q2: 100,
			},
			list: [],
			drawer: false,
		}),
		computed: {
			tree() {
				return this.list.map((e) => {
					return e.tree;
				});
			},
		},

		methods: {
			test() {
				console.log('clci');

				let node = new Node(0, [1, 3, 5, 4]);
				this.list.push({ tree: node });
				node = new Node(1, [0, 2]);
				this.list.push({ tree: node });
				node = new Node(2, [1, 5]);
				this.list.push({ tree: node });
				node = new Node(3, [1]);
				this.list.push({ tree: node });
				node = new Node(4, [1]);
				this.list.push({ tree: node });
				node = new Node(5, [1, 2]);
				this.list.push({ tree: node });
			},
			NodesToArrayIdNot(s) {
				return this.tree
					.filter((e) => {
						return e.id != s;
					})
					.map((e) => {
						return e.id;
					});
			},
			NodesToArrayId() {
				return this.tree.map((e) => {
					return e.id;
				});
			},
			select(e, i) {
				this.tree
					.filter((t) => t.id != i && e.indexOf(t.id) === -1)
					.map((t) => {
						t.child = t.child.filter((tt) => {
							return tt != i;
						});
						return t;
					});
				if (e.length > 0 && this.tree.filter((t) => t.id == e[e.length - 1])[0].child.indexOf(i) === -1) {
					this.tree.filter((t) => t.id == e[e.length - 1])[0].child.push(i);
				}
			},
			nodes() {
				let nodes = [];
				this.tree.forEach((e) => {
					nodes.push(this.getRow(e.id));
				});
				return nodes;
			},
			getRow(id) {
				let ar = new Array(this.tree.length).fill(0);
				ar[id] = null;
				this.tree[id].child.forEach((e) => {
					ar[e] = 1;
				});
				return ar;
			},
			add() {
				let n = Math.max.apply(null, this.NodesToArrayId()) + 1;
				if (!isFinite(n)) n = 0;
				let node = new Node(n, []);
				this.list.push({ tree: node });
			},
			save() {
				localStorage.nodes = JSON.stringify(this.nodes());
			},
			del(id) {
				this.list.splice(
					this.tree.findIndex((e) => {
						return e.id == id;
					}),
					1,
				);

				this.list.forEach((e) => {
					e.tree.child = e.tree.child.filter((t) => {
						return t != id;
					});
				});
			},
		},
	};
	class Node {
		constructor(id, c) {
			this.id = id;
			this.child = c;
		}
	}
</script>
