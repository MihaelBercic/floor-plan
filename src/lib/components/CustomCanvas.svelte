<script lang="ts">
    import {onMount} from "svelte";
    import {fade} from "svelte/transition";
    import {
        ActionState,
        type CanvasElement,
        CanvasElementType,
        currentActionState,
        currentLinesStore,
        currentlySelectedObject,
        GridPosition,
        type Line,
        type Point2D,
        type Room
    } from "$lib/types/Graphics";
    import Polygon from "polygon";

    let mouseDown: boolean = false;
    let toolbar: HTMLElement | undefined
    export let debug: boolean = true

    const objects: CanvasElement[] = [{
        type: CanvasElementType.Door,
        position: {x: 0, y: 0},
        rotation: 0
    }];

    const testRoom: Room = {
        quality: "good",
        lines: [
            {start: {x: -5, y: -5}, end: {x: -5, y: 5}},
            {start: {x: -5, y: 5}, end: {x: 5, y: 5}},
            {start: {x: 5, y: 5}, end: {x: 5, y: -5}},
            {start: {x: 5, y: -5}, end: {x: -5, y: -5}},
        ],
        position: {x: 0, y: 0},
        rotation: 0,
        type: CanvasElementType.Room
    }
    let rooms: Room[] = [testRoom];

    let canvas: HTMLCanvasElement;
    let ctx: CanvasRenderingContext2D;
    let windowHeight: number, windowWidth: number

    const minimumPointSize = 20
    let lineWidth: number;
    let scale = 1.0;
    let pointSize = minimumPointSize
    let halfPointSize: number
    $: halfPointSize = pointSize / 2
    $: lineWidth = 1 / 3 * pointSize

    let centerPosition: Point2D = {x: 0, y: 0}
    let canvasOffset: Point2D = {x: 0, y: 0}
    let mousePosition: Point2D = centerPosition
    let moveStartPosition: Point2D = centerPosition

    function setCenter() {
        let clientWidth = canvas.clientWidth
        let clientHeight = canvas.clientHeight
        centerPosition = {x: clientWidth / 2, y: clientHeight / 2}
    }


    function move(event: MouseEvent) {
        mousePosition = {x: event.clientX, y: event.clientY}
        if (event.button == 1 || 1 == event.button & 2) { // Middle button = 1, Left = 0, Right = 2
            let distanceX = event.clientX - moveStartPosition.x
            let distanceY = event.clientY - moveStartPosition.y

            canvasOffset.x += distanceX
            canvasOffset.y += distanceY
            moveStartPosition = {...mousePosition}
        }
        if (mouseDown) onDrag()
    }

    function redraw() {
        if (ctx) {
            ctx.clearRect(0, 0, 10000, 1000)
            setCenter()
            drawGrid()
            rooms.forEach(drawRoom)
            drawObjects()

            const first = $currentLinesStore[0]
            if (first) {
                ctx.strokeStyle = "orange"
                ctx.lineWidth = lineWidth
                const transformed = transformFakeToDrawable(first.start);
                ctx.beginPath()
                ctx.moveTo(transformed.x, transformed.y)
                $currentLinesStore.forEach((line) => drawLine(line))
                ctx.stroke()
            }

            ctx.strokeStyle = "blue"
            const mouseGridCoordinates: Point2D = transformRealToFake(mousePosition)
            const drawableMouseCoordinates = transformFakeToDrawable(mouseGridCoordinates)

            const transformedX = mousePosition.x - centerPosition.x
            const transformedY = centerPosition.y - mousePosition.y
            ctx.lineWidth = 1
            ctx.strokeText(`C: ${Math.round(canvasOffset.x)}px, ${Math.round(canvasOffset.y)}px`, mousePosition.x, mousePosition.y + 70)
            ctx.strokeText(`A: ${mouseGridCoordinates.x}, ${mouseGridCoordinates.y}`, mousePosition.x, mousePosition.y + 60)
            ctx.strokeText(`T: ${transformedX}, ${transformedY}`, mousePosition.x, mousePosition.y + 50)
            ctx.strokeText(`R: ${mousePosition.x}, ${mousePosition.y}`, mousePosition.x, mousePosition.y + 40)

            ctx.fillStyle = "rgba(150, 30, 20, .1)"
            ctx.beginPath()
            ctx.arc(drawableMouseCoordinates.x, drawableMouseCoordinates.y, pointSize, 0, 360)
            ctx.closePath()
            ctx.fill()
        }
    }

    function zoom(e: WheelEvent) {
        const distanceCenterX = centerPosition.x - mousePosition.x
        const delta = e.deltaY
        scale += delta / 100
        scale = Math.max(0.5, Math.min(5, scale));
        pointSize = minimumPointSize * scale;
        canvasOffset.x += distanceCenterX / 10 * Math.sign(delta);
    }

    function drawGrid() {
        const numberOfColumns = windowWidth / pointSize
        const numberOfRows = windowHeight / pointSize
        const lineWidth = pointSize / 100;
        ctx.lineWidth = lineWidth
        ctx.strokeStyle = `rgba(0,0,0, ${scale / 10})`;
        //ctx.strokeStyle = `rgba(0,0,0, 0)`;
        for (let i = -windowHeight; i < windowWidth; i++) {
            ctx.beginPath()
            const x = i * pointSize + canvasOffset.x + halfPointSize - lineWidth / 2
            ctx.strokeText(`${i}`, x - pointSize * 5 / 8, 50)
            ctx.moveTo(x, 0);
            ctx.lineTo(x, windowHeight);
            ctx.closePath()
            ctx.stroke()
        }
        for (let i = -windowWidth; i < windowHeight; i++) {
            const y = -i * pointSize + canvasOffset.y - halfPointSize + lineWidth / 2
            ctx.strokeText(`${i}`, 50, y + pointSize * 5 / 8)
            ctx.beginPath()
            ctx.moveTo(0, y);
            ctx.lineTo(windowWidth, y);
            ctx.closePath()
            ctx.stroke()
        }

    }

    const transformFakeToDrawable = (point: Point2D, xPosition: GridPosition = GridPosition.Center, yPosition: GridPosition = GridPosition.Center) => {
        let addX = 0, addY = 0;
        if (xPosition == GridPosition.Right) addX = pointSize;
        if (xPosition == GridPosition.Center) addX = halfPointSize;

        if (yPosition == GridPosition.Bottom) addY = pointSize
        if (yPosition == GridPosition.Center) addY = halfPointSize
        return {x: (point.x * pointSize + canvasOffset.x - halfPointSize) + addX, y: (-point.y * pointSize + canvasOffset.y) - halfPointSize + addY}
    }

    const transformRealToFake = (point: Point2D) => {
        const absolutePosition = {x: point.x - canvasOffset.x, y: canvasOffset.y - point.y}
        return {x: Math.round(absolutePosition.x / pointSize), y: Math.round(absolutePosition.y / pointSize)}
    }

    onMount(async () => {
        dpiFix()
        canvasOffset = centerPosition
    })

    $: if (windowHeight) dpiFix()
    $: if (windowWidth) dpiFix()
    $: if (mousePosition) redraw()
    $: if (mousePosition) drawCurrentLine()
    $: if (pointSize) redraw()
    $: if ($currentActionState) redraw()
    $: if ($currentlySelectedObject) redraw()

    function drawRoom(room: Room) {
        colorRoom(room)
        ctx.beginPath()
        room.lines.forEach(drawLine)
        ctx.strokeStyle = "black"
        ctx.lineWidth = lineWidth
        ctx.stroke();
        const polygon = new Polygon(room.lines.map(x => x.start));
        const polygonArea = polygon.area()
        ctx.fillStyle = "white"
        room.lines.forEach((line) => displayLineDistance(line, undefined, polygon))
        const polygonCenter = polygon.center()
        const roomCenter = {x: polygonCenter.x, y: polygonCenter.y}
        const transformed = transformFakeToDrawable(roomCenter);
        const text = `${Math.abs(polygonArea / (pointSize * pointSize) * 100).toFixed(1)}m`;
        const metrics = ctx.measureText(text);
        ctx.fillText(text, transformed.x, transformed.y)
        ctx.font = `${halfPointSize * 0.8}px Arial`
        ctx.fillText("2", transformed.x + metrics.actualBoundingBoxRight + 5, transformed.y - metrics.fontBoundingBoxAscent)
    }

    function colorRoom(room: Room) {
        if (room.lines.length <= 2) return
        const startPoint = room.lines[0].start;
        const lastPoint = room.lines[room.lines.length - 2].start;
        const transformedStartPoint = transformFakeToDrawable(startPoint);
        const transformedLastPoint = transformFakeToDrawable(lastPoint);
        const grd = ctx.createLinearGradient(transformedStartPoint.x, transformedStartPoint!.y, transformedLastPoint.x, transformedLastPoint.y)
        if (room.quality === "good") {
            grd.addColorStop(0, "rgba(30, 255, 150, 0.8)");
            grd.addColorStop(1, "rgba(30, 200, 100, 0.8)");
        } else {
            grd.addColorStop(0, "rgb(218,26,26)");
            grd.addColorStop(1, "rgb(200,100,127)");
        }
        ctx.fillStyle = grd
        ctx.beginPath()
        ctx.moveTo(transformedStartPoint.x, transformedStartPoint.y)
        room.lines.forEach(({start, end}) => {
            const transformedEnd = transformFakeToDrawable(end)
            ctx.lineTo(transformedEnd.x, transformedEnd.y)
        })
        ctx.moveTo(transformedStartPoint.x, transformedStartPoint.y)
        ctx.closePath()
        ctx.fill()
    }

    function drawLine(line: Line) {
        const {start, end} = line;
        const direction = {x: end.x - start.x, y: end.y - start.y}
        let xPos = GridPosition.Left, yPos = GridPosition.Top;
        if (direction.x > 0 || direction.y < 0) xPos = GridPosition.Right;
        if (direction.x < 0) xPos = GridPosition.Left;
        if (direction.y > 0) yPos = GridPosition.Top;
        if (direction.y < 0 || direction.x < 0) yPos = GridPosition.Bottom;

        const startTransformed = transformFakeToDrawable(line.start)
        const endTransformed = transformFakeToDrawable(line.end)
        ctx.lineCap = "round"
        ctx.moveTo(startTransformed.x, startTransformed.y)
        ctx.lineTo(endTransformed.x, endTransformed.y);
        // ctx.fillText(`${JSON.stringify(direction)} = ${xPos, yPos}`, endTransformed.x + 200, endTransformed.y + 0.1 * direction.y)
    }

    function displayLineDistance(line: Line, textPosition: Point2D | undefined = undefined, polygon: Polygon | undefined = undefined) {
        const {start, end} = line
        let direction = getLineDirection(line);
        let xAdd = 0, yAdd = 0;
        const distance = Math.sqrt(Math.abs((start.x - end.x)) ** 2 + Math.abs(start.y - end.y) ** 2);
        const centerPoint = {x: (end.x + start.x) / 2, y: (end.y + start.y) / 2}
        const transformedCenter = transformFakeToDrawable(centerPoint);

        if (polygon) {
            const {x, y} = polygon.center();
            const polyCenter: Point2D = {x: x, y: y}
            direction = getLineDirection({start: centerPoint, end: polyCenter})
        }
        xAdd = direction.x === 0 ? 0 : Math.sign(direction.x) * pointSize;
        yAdd = direction.y === 0 ? 0 : -Math.sign(direction.y) * halfPointSize * 1.25;

        ctx.font = `${halfPointSize}px Verdana`;
        ctx.textAlign = "center"
        ctx.textBaseline = "middle"
        ctx.fillText(`${(distance / 2).toFixed(1)}m`, textPosition?.x ?? (transformedCenter.x + xAdd), textPosition?.y ?? (transformedCenter.y + yAdd))
    }

    function dpiFix() {
        ctx = canvas.getContext("2d")
        const width = windowWidth
        const height = windowHeight
        const ratio = Math.ceil(window.devicePixelRatio);
        canvas.width = width * ratio;
        canvas.height = height * ratio;
        canvas.style.width = `${width}px`;
        canvas.style.height = `${height}px`;
        ctx.scale(ratio, ratio)
        setCenter()
        redraw()
    }

    function onClick(event: MouseEvent) {
        const gridPosition = transformRealToFake({x: event.clientX, y: event.clientY})
        if (event.movementX > 0 || event.movementY > 0) return;
        if ($currentActionState === ActionState.Drawing) {
            const linesCount = $currentLinesStore.length
            const lastLine = $currentLinesStore[linesCount - 1];
            const existingLine = lastLine ?? {start: {x: gridPosition.x, y: gridPosition.y}, end: {x: gridPosition.x, y: gridPosition.y}}
            console.log($currentLinesStore.length)
            if (!$currentLinesStore.includes(existingLine)) {
                currentLinesStore.update(lines => {
                    lines.push(existingLine);
                    return lines;
                })
            }
            if (JSON.stringify(gridPosition) !== JSON.stringify(existingLine.start)) {
                existingLine.end = gridPosition;
                if (JSON.stringify(existingLine.end) === JSON.stringify($currentLinesStore[0].start)) {
                    const polygon = new Polygon($currentLinesStore.map(x => x.end));
                    const center = polygon.center()
                    const newRoom: Room = {
                        lines: [...$currentLinesStore],
                        quality: Math.random() > 0.5 ? "good" : "bad",
                        position: {x: center.x, y: center.y},
                        rotation: 0,
                        type: CanvasElementType.Door,
                    }
                    $currentLinesStore = []
                    $currentActionState = ActionState.None
                    rooms = [...rooms, newRoom]
                } else {
                    currentLinesStore.update(lines => {
                        lines.push({start: {x: gridPosition.x, y: gridPosition.y}, end: {x: gridPosition.x, y: gridPosition.y}});
                        return lines;
                    })
                }
                return
            }
        } else {
            const area = [
                {x: gridPosition.x - 2, y: gridPosition.y - 2},
                {x: gridPosition.x - 2, y: gridPosition.y + 2},
                {x: gridPosition.x + 2, y: gridPosition.y + 2},
                {x: gridPosition.x + 2, y: gridPosition.y - 2},
                {x: gridPosition.x - 2, y: gridPosition.y - 2},
            ]
            const polygon = new Polygon(area);
            const firstObjectOnGridPosition = objects.find(x => polygon.contains({x: x.position.x, y: x.position.y, w: 0, h: 0}))
            $currentlySelectedObject = firstObjectOnGridPosition;
        }
    }

    function getLineDirection(line: Line) {
        const {start, end} = line;
        return {x: end.x - start.x, y: end.y - start.y};
    }

    function drawCurrentLine() {
        const line = $currentLinesStore[$currentLinesStore.length - 1];
        if (ctx && line) {
            const direction = getLineDirection(line);
            const transformedStart = transformFakeToDrawable(line.start);
            line.end = transformRealToFake(mousePosition);
            ctx.beginPath()
            drawLine(line);
            ctx.lineWidth = lineWidth
            ctx.stroke()
            ctx.fillStyle = "black"
            displayLineDistance(line, {x: mousePosition.x + pointSize, y: mousePosition.y - pointSize});
        }
    }

    let toolbarPosition: Point2D = {x: 0, y: 0}

    function drawObjects() {
        for (let canvasObject of objects) {
            if (canvasObject.type === CanvasElementType.Door) {
                const doorCenter = transformFakeToDrawable(canvasObject.position)
                const rotation = canvasObject.rotation
                toolbarPosition = doorCenter
                ctx.strokeStyle = $currentlySelectedObject === canvasObject ? "orange" : "black";
                ctx.lineWidth = pointSize * ($currentlySelectedObject === canvasObject ? 0.2 : 0.15);
                ctx.beginPath()
                ctx.moveTo(doorCenter.x, doorCenter.y)
                ctx.arc(doorCenter.x, doorCenter.y, 2 * pointSize, Math.PI * (rotation - 0.5), Math.PI * rotation)
                ctx.stroke()
                ctx.strokeStyle = "white"
                ctx.lineWidth = lineWidth
                ctx.lineCap = "square"
                ctx.beginPath()
                ctx.moveTo(doorCenter.x, doorCenter.y);
                ctx.lineTo(doorCenter.x + 2 * pointSize, doorCenter.y);
                ctx.stroke()
            }
        }
    }

    function onDrag() {
        const currentObject = $currentlySelectedObject;
        if (!currentObject) return;
        const drawableMousePosition = transformRealToFake(mousePosition)
        currentObject.position = drawableMousePosition
    }

</script>
<svelte:window bind:innerHeight={windowHeight} bind:innerWidth={windowWidth}/>
<canvas
        bind:this={canvas}
        on:click={onClick}
        on:mousedown={e =>{
            moveStartPosition = {x: e.clientX, y: e.clientY}
            mouseDown = e.button === 0;
        }}
        on:mouseenter={e => (moveStartPosition = {x: e.clientX, y: e.clientY})}
        on:mousemove={move}
        on:mouseup={() => {mouseDown = false}}
/>
{#if toolbarPosition && $currentlySelectedObject}
    <div in:fade out:fade
         bind:this={toolbar} id="toolbar"
         style="left: {toolbarPosition.x - ((toolbar?.clientWidth / 2) ?? 0)}px; top: {toolbarPosition.y + 2* pointSize}px; transition: none">
        <div class="button"><img src="/rotate.svg" on:click={() =>($currentlySelectedObject.rotation += 0.5)}></div>
        <div class="button"><img src="/trash.svg"></div>
    </div>
{/if}
<style>
    canvas {
        position: relative;
        flex: 1;
    }

    #toolbar {
        position: absolute;
        display: flex;
        gap: .5em;
    }

    #toolbar > .button {
        display: flex;
        justify-content: center;
        align-items: center;
        background-color: white;
        box-shadow: 0 0 5px 5px rgba(0, 0, 0, 0.05);
        border-radius: 5px;
    }

    .button img {
        width: 1em;
        height: 1em;
        padding: .5em;
    }

    #toolbar > .button:hover {
        transform: scale(1.1);
        cursor: pointer;
    }
</style>