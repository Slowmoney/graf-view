<template>
	<v-container>
		<canvas ref="canvas"></canvas>
		{{ nodes() }}::: {{ n }}
		<!-- это не удалять запускает Update -->
	</v-container>
</template>

<script>
	export default {
		name: 'Main',
		data() {
			return {
				Toloop: 0,
				dt: 0.004,
				size: 800,
				vmax: 80,
				frames: 1000,
				/* 	params: {
					gamma: 2,
					l0: 200,
					k: 1,
					q2: 100,
				}, */
			};
		},
		props: {
			reload: Boolean,
			tree: Array,
			params: Object,
		},
		beforeCreate() {},
		created() {
			if (this.Toloop == 0) {
				requestAnimationFrame(this.loop);
				console.log('lopp');
			}
		},
		mounted() {
			console.log('mounted');

			this.canvas = new Canvas(this.$refs.canvas);

			this.update();
		},
		beforeUpdate() {
			console.log('beforeUpdate');
			this.update();
		},
		updated() {
			console.log('updated');
		},
		watch: {
			params: {
				handler: function() {
					this.update();
				},
				deep: true,
			},
			reload() {
				this.update();
			},
		},
		methods: {
			update() {
				this.graph = new Graph(this.nodes());
				let r = this.rndPoss(this.n, this.size / 2, true);
				let v = this.rndPoss(this.n, this.vmax, false);
				this.ps = new PhysSys(this.model(), this.dt).init(r, v);
			},
			model() {
				return this.ForceFrictionModel(this.LogSpringChargeForce(this.graph, this.params), this.SquareForce(this.params.gamma));
			},
			nodes() {
				let nodes = [];

				this.tree.forEach((e, i) => {
					nodes.push(this.getRow(e.id, i));
				});
				this.n = nodes.length;
				/* console.log('nodes', nodes); */

				return nodes;
			},
			getRow(id, i) {
				let ar = new Array(this.tree.length).fill(0);
				this.tree[i].child.forEach((e) => {
					ar[this.tree.findIndex((tre) => tre.id == e)] = 1;
				});
				ar[i] = null;
				return ar;
			},
			EdgeNodeForce(graph, edge, noedge) {
				let n = graph.getSize();
				return function(r) {
					let f = new Array(2 * n);
					let i;
					for (i = 0; i < n; ++i) f[2 * i] = f[2 * i + 1] = 0;
					for (i = 0; i < n; ++i)
						for (let j = i + 1; j < n; ++j) {
							let dx = r[2 * i] - r[2 * j];
							let dy = r[2 * i + 1] - r[2 * j + 1];
							let dl = Math.sqrt(dx * dx + dy * dy);
							if (dl == 0) continue;
							let fs = graph.edge(i, j) ? edge(dl) : noedge(dl);
							let fx = (fs * dx) / dl;
							let fy = (fs * dy) / dl;
							f[2 * i] += fx;
							f[2 * j] -= fx;
							f[2 * i + 1] += fy;
							f[2 * j + 1] -= fy;
						}
					return f;
				};
			},
			ForceFrictionModel(force, friction) {
				return function(r, v) {
					var n = r.length / 2;
					var f = force(r);
					for (var i = 0; i < n; ++i) {
						var dv = Math.sqrt(v[2 * i] * v[2 * i] + v[2 * i + 1] * v[2 * i + 1]);
						if (dv == 0) continue;
						f[2 * i] -= (friction(dv) * v[2 * i]) / dv;
						f[2 * i + 1] -= (friction(dv) * v[2 * i + 1]) / dv;
					}
					return f;
				};
			},
			LogSpringChargeForce(graph, params) {
				let l0 = params.l0,
					k = params.k,
					q2 = params.q2;
				return this.EdgeNodeForce(
					graph,
					function(dl) {
						return q2 / (dl * dl) - k * Math.log(dl / l0);
					},
					function(dl) {
						return q2 / (dl * dl);
					},
				);
			},
			SquareForce(gamma) {
				return function(v) {
					return gamma * v * v;
				};
			},
			centralize(r, x0, y0) {
				var xm = 0,
					ym = 0;
				var n = r.length / 2;
				for (var i = 0; i < n; ++i) {
					xm += r[2 * i];
					ym += r[2 * i + 1];
				}
				xm = xm / n;
				ym = ym / n;
				var a = [].concat(r);
				for (var j = 0; j < n; ++j) {
					a[2 * j] += x0 - xm;
					a[2 * j + 1] += y0 - ym;
				}
				return a;
			},
			rndPoss(n, mx, sym) {
				var r = new Array(2 * n);
				for (var i = 0; i < 2 * n; ++i)
					if (sym) {
						r[i] = Math.random() * mx;
					} else {
						r[i] = Math.random() * 2 * mx - mx;
					}
				return r;
			},
			stablePoint(physsys, params) {
				if (!params) params = { K0: 1e-6, dK: 1e-2, a: 3.0 / 4.0, b: 3.0 };
				var K0 = params.K0, // value of final kin energy
					dK = params.dK, // value of magnitude of kin energy fluctuation (last maximum)
					a = params.a, // fraction of maximums to skip
					b = params.b; // kin energy limitation b*dK
				if (!K0 || !dK || !a || !b) throw 'invalid parameters';
				physsys.step();
				var k2 = physsys.kinEnergy();
				physsys.step();
				var k1 = physsys.kinEnergy();
				var K = [];
				var cnd = true;
				do {
					physsys.step();
					var k = physsys.kinEnergy();
					if (k1 > k2 && k1 > k) {
						K.push(k1);
						cnd = cond();
					}
					k2 = k1;
					k1 = k;
				} while (cnd || k1 > K0 || K[K.length - 1] > dK);

				function cond() {
					for (var i = Math.floor(K.length * a); i < K.length; ++i) if (K[i] > b * dK) return true;
					return false;
				}

				return physsys.state();
			},
			loop() {
				this.Toloop = 1;
				this.canvas.clear();

				for (let i = 0; i < this.frames; ++i) this.ps.step();
				let po = this.centralize(this.ps.state(), this.size / 2, this.size / 2);
				for (let i = 0; i < this.n; ++i) for (let j = i + 1; j < this.n; ++j) if (this.graph.edge(i, j)) this.canvas.drawLine(po[2 * i], po[2 * i + 1], po[2 * j], po[2 * j + 1]);
				for (let i = 0; i < po.length / 2; i++) {
					this.canvas.ctx.save();
					this.canvas.ctx.shadowBlur = 10;
					this.canvas.ctx.shadowColor = '#333';
					this.canvas.drawCircle(po[2 * i], po[2 * i + 1], 20, '#000');
					this.canvas.ctx.restore();
					this.canvas.drawText(po[2 * i], po[2 * i + 1] + 8, this.tree[i].id);
				}

				requestAnimationFrame(this.loop);
			},
		},
	};
	class Canvas {
		constructor(id, width = 1024, height = 1024) {
			this.canvas = id;
			/** @type {CanvasRenderingContext2D} */
			this.ctx = this.canvas.getContext('2d');

			this.width = this.canvas.width = width;
			this.height = this.canvas.height = height;
		}
		drawBox = (options /* = { x, y, color, c, szx, szy, s, n } */) => {
			this.ctx.save();

			this.ctx.beginPath();

			this.ctx.rect(options.x, options.y, options.szx, options.szy);

			if (options.c) {
				this.ctx.fillStyle = options.color;
				this.ctx.fill();
			}
			if (options.s) {
				this.ctx.stroke();
			}
			this.ctx.addHitRegion({ id: [options.n, this.name] });
			this.ctx.restore();
		};
		drawCircle = (x, y, r, color = 0) => {
			this.ctx.save();
			this.ctx.beginPath();
			this.ctx.arc(x, y, r, 0, 2 * Math.PI);
			/* 	let gradient = this.ctx.createRadialGradient(x, y, 1, x, y, r + 30);

			gradient.addColorStop(0, 'rgba(255,255,255,1)');
			gradient.addColorStop(0.53, 'rgba(0,0,0,1)');
			gradient.addColorStop(1, 'rgba(0,0,0,1)'); */
			/* this.ctx.fillStyle = gradient; */
			/* this.ctx.strokeStyle = gradient; */
			this.ctx.fillStyle = color;
			this.ctx.strokeStyle = color;

			this.ctx.fill();
			/* this.ctx.stroke(); */
			this.ctx.restore();
		};
		drawLine = (x0, y0, x1, y1, color) => {
			this.ctx.save();
			this.ctx.beginPath();
			this.ctx.moveTo(x0, y0);
			this.ctx.lineTo(x1, y1);
			this.ctx.strokeStyle = color;
			this.ctx.stroke();
			this.ctx.restore();
		};
		drawText = (x, y, t) => {
			this.ctx.font = '20px arial';
			this.ctx.textAlign = 'center';
			this.ctx.fillStyle = '#fff';
			this.ctx.fillText(t, x, y);
		};
		clear = () => {
			this.ctx.clearRect(0, 0, this.width, this.height);
		};
	}
	class Graph {
		constructor(adj) {
			this.adj = adj;
			this.n = this.adj.length;
			/* if (!this.n) {
					this.n = this.adj.length;
				} */
		}
		getAdjacencyMatrix = function() {
			return this.adj;
		};
		getSize = function() {
			return this.n;
		};
		edge = function(i, j) {
			return this.adj[i][j];
		};
		checkAdjacencyMatrix(a) {
			if (!a || !a.length) return false;
			var n = a.length;
			for (var i = 0; i < n; ++i) if (!a[i] || !(a[i].length === n)) return false;
			return true;
		}
		connected() {
			var adj = this.getAdjacencyMatrix();
			var n = this.getSize();
			var labled = [];
			var queue = [0];
			do {
				let start = queue.pop();
				for (var i = 0; i < n; ++i)
					if (!labled[i] && adj[start][i]) {
						labled[i] = true;
						queue.push(i);
					}
			} while (queue.length > 0);

			for (let i = 0; i < n; ++i) if (!labled[i]) return false;
			return true;
		}
	}
	class PhysSys {
		constructor(f, dt) {
			this.f = f;
			this.dt = dt;
		}
		init = (r, v) => {
			r = [].concat(r);
			v = [].concat(v);
			let dt = this.dt;

			function add(x, dx) {
				var n = x.length;
				for (var i = 0; i < n; ++i) x[i] += dx[i] * dt;
			}
			var f = this.f;

			return {
				step: () => {
					/* console.log('step', f, r, v); */

					var fv = f(r, v);
					/* console.log('step', fv); */
					add(v, fv);
					add(r, v);
				},
				state: () => {
					return r;
				},
				kinEnergy: () => {
					var E = 0;
					for (var i = 0; i < v.length; ++i) E += v[i] * v[i];
					return E / 2;
				},
			};
		};
	}
</script>
