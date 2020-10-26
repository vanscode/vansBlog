<template>
    <button @click="handleClick">我是一个按钮</button>
</template>
<script>
    export default {
        method: {
            handleClick() {
                console.log("我是一个按钮")
            }
        }
    }
</script>
