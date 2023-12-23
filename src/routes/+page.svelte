<script lang="ts">
	import { onMount } from 'svelte';
	import { fabric } from 'fabric';
	import party from 'party-js';
	let rouletteBoardCanvas: HTMLCanvasElement;
	let canvas: HTMLCanvasElement;
	let fabricCanvas: fabric.Canvas;

	let arrowImage: fabric.Image;

	let isStarted: boolean = false;
	let speed: number;
	let acceleration: number = 0;

	let resultText = '';

	/**
	 * min以上max以下の整数をランダムに返す
	 * @param min 最小値
	 * @param max 最大値
	 */
	const getRand = (min: number, max: number) => {
		return Math.floor(Math.random() * (max - min + 1) + min);
	};

	const CANVAS_WIDTH = 480;
	const CANVAS_HEIGHT = 480;
	const CENTER = {
		X: CANVAS_WIDTH / 2,
		Y: CANVAS_HEIGHT / 2
	};
	const RESULT_COUNT = 100;
	const DEAD_NUMBER = getRand(0, RESULT_COUNT - 1);

	onMount(() => {
		// ルーレット盤を描画
		drawRouletteBoard();
		// Fabric.jsのCanvasを初期化
		initializeFabricCanvas();
	});

	/**
	 * ルーレット盤を描画
	 */
	const drawRouletteBoard = () => {
		const context = rouletteBoardCanvas.getContext('2d')!;
		const RESULT_ANGLE = 360 / RESULT_COUNT;
		const CENTER = {
			X: CANVAS_WIDTH / 2,
			Y: CANVAS_HEIGHT / 2
		};
		[...Array(RESULT_COUNT)].map((_, i) => {
			context.beginPath();
			context.moveTo(CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2);
			context.fillStyle = i % 2 === 0 ? '#666666' : '#999999';
			const startAngle = (RESULT_ANGLE * (i + 1) * Math.PI) / 180;
			const endAngle = (RESULT_ANGLE * i * Math.PI) / 180;
			context.arc(CENTER.X, CENTER.Y, CANVAS_WIDTH / 2, startAngle, endAngle, true);
			context.fill();
		});
	};

	/**
	 * Fabric.jsのCanvasを初期化
	 */
	const initializeFabricCanvas = async () => {
		const rouletteBoardBase64Src: string = rouletteBoardCanvas.toDataURL('image/png');

		fabricCanvas = new fabric.Canvas(canvas, {
			containerClass: 'fabric'
		});
		fabricCanvas.setBackgroundImage(rouletteBoardBase64Src, () => {
			fabricCanvas.renderAll();

			drawRouletteTexts();
			const backgroundBase64Sec = fabricCanvas.toDataURL();
			fabricCanvas.clear();
			fabricCanvas.setBackgroundImage(backgroundBase64Sec, () => {
				fabricCanvas.renderAll();
			});

			drawArrow();
		});
	};

	/**
	 * ルーレットのテキストを描画
	 */
	const drawRouletteTexts = () => {
		const UNIT_ANGLE = 360 / RESULT_COUNT;
		const RADIUS = CANVAS_WIDTH / 2;
		[...Array(RESULT_COUNT)].map((_, i) => {
			const degree = (i + 0.5) * UNIT_ANGLE;
			const radian = (degree - 270) * (Math.PI / 180);
			const isDeadNumber = () => {
				return i === DEAD_NUMBER;
			};
			const text = isDeadNumber() ? '死' : '勝\n利';
			const textColor = isDeadNumber() ? '#ff0000' : '#111111';
			const element = new fabric.Text(text, {
				left: Math.cos(radian) * RADIUS * 0.9 + CENTER.X,
				top: Math.sin(radian) * RADIUS * 0.9 + CENTER.X,
				originX: 'center',
				originY: 'center',
				fill: textColor,
				fontSize: 12,
				fontFamily: 'Ariel',
				fontWeight: 'bold',
				angle: degree,
				selectable: false,
				objectCaching: true
			});
			fabricCanvas.add(element);
		});
	};

	/**
	 * 矢印を描画
	 */
	const drawArrow = () => {
		fabric.Image.fromURL('/arrow.png', (img) => {
			img.selectable = false;
			img.originX = 'left';
			img.originY = 'center';
			img.left = CENTER.X;
			img.top = CENTER.Y;
			img.scale(0.5);
			img.centeredRotation = false;
			arrowImage = img;
			fabricCanvas.add(arrowImage);
			drawHead();
		});
	};

	/**
	 * ルーレット盤中央の頭部分を描画
	 */
	const drawHead = () => {
		const circle = new fabric.Circle({
			originX: 'center',
			originY: 'center',
			left: CENTER.X,
			top: CENTER.Y,
			selectable: false,
			radius: 15,
			fill: '#ffff00',
			borderColor: '#9F8830'
		});
		fabricCanvas.add(circle);
	};

	/**
	 * 針回転開始
	 */
	const startRotate = () => {
		if (isStarted) {
			return;
		}
		resultText = '';
		isStarted = true;
		acceleration = 0;
		const MAX_SPEED = 17;
		speed = MAX_SPEED;
		move();
	};

	/**
	 * 針回転
	 */
	const move = () => {
		const MIN_SPEED = 0.1;
		speed += acceleration;
		arrowImage.rotate(arrowImage.angle! + speed);
		fabricCanvas.renderAll();

		if (speed <= MIN_SPEED) {
			speed = 0;
			isStarted = false;
			if (isWin()) {
				resultText = '勝利';
				party.confetti(document.body, {
					count: party.variation.range(80, 100),
					spread: 40,
					speed: party.variation.range(50, 600),
					size: party.variation.skew(1, 0.8),
					rotation: () => party.random.randomUnitVector().scale(180)
				});
			} else {
				resultText = '死';
			}
			return;
		}

		fabric.util.requestAnimFrame(move);
	};

	/**
	 * 針回転停止
	 */
	const stopRotate = () => {
		if (acceleration !== 0) {
			return;
		}
		acceleration = -0.1;
	};

	/**
	 * 勝利判定
	 * @returns 勝利したかどうか
	 */
	const isWin = () => {
		const deadAngleFrom = (90 + (DEAD_NUMBER * 360) / RESULT_COUNT) % 360;
		const deadAngleTo = (90 + ((DEAD_NUMBER + 1) * 360) / RESULT_COUNT) % 360;
		const angle = arrowImage.angle! % 360;
		arrowImage.rotate(angle);
		fabricCanvas.renderAll();
		return deadAngleFrom > angle || deadAngleTo < angle;
	};

	/**
	 * ツイート
	 */
	const tweet = () => {
		const tweetURL = `https://twitter.com/intent/tweet?text=`;
		const message = `${resultText}!!`;
		const tweetText = `${message}\n\n#Web戦吉兆占針盤\n${location.origin}`;
		const encodedText = tweetURL + encodeURIComponent(tweetText);

		window.open(encodedText, '_blank', 'noreferrer');
	};
</script>

<div class="container">
	<img src="/title.png" alt="title" />
	<p class="description">
		魁!!男塾に登場するルーレットを再現したものです。
		<br />大勝負の前の景気づけにどうぞ。
	</p>
	<canvas
		bind:this={rouletteBoardCanvas}
		width={CANVAS_WIDTH}
		height={CANVAS_HEIGHT}
		class="hidden"
	>
	</canvas>
	<canvas bind:this={canvas} width={CANVAS_WIDTH} height={CANVAS_HEIGHT}></canvas>
	{#if !isStarted}
		<button class="btn" on:click={startRotate}>
			<img src="/start.png" alt="start" />
		</button>
	{:else}
		<button class="btn" on:click={stopRotate}>
			<img src="/stop.png" alt="stop" />
		</button>
	{/if}
	{#if resultText !== ''}
		<button class="btn tweet" on:click={tweet}>
			<img src="/tweet.png" alt="tweet" />
		</button>
	{/if}
</div>

<style>
	.hidden {
		display: none !important;
	}

	img {
		max-width: 100%;
		height: 75px;
	}

	canvas {
		max-width: 100%;
	}

	.btn {
		margin-top: 20px;
		background-color: rgb(47, 79, 79);
		border: none;
		width: 300px;
		max-width: 100%;
	}
	.btn:hover {
		cursor: pointer;
		background-color: rgb(20, 50, 50);
	}
	.btn img {
		height: 45px;
	}

	.tweet {
		background-color: rgb(85, 172, 238);
	}
	.tweet:hover {
		background-color: rgb(65, 152, 218);
	}

	.container {
		display: flex;
		flex-direction: column;
		justify-content: flex-start;
		align-items: center;
	}

	.description {
		font-weight: bold;
		color: white;
		margin-top: 0px;
		margin-bottom: 5px;
	}
</style>
