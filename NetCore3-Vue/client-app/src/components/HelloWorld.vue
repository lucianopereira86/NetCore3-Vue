<template>
    <div>
        <div :key="r" v-for="r in this.results">{{ r }}</div>
    </div>
</template>

<script>
export default {
        name: 'HelloWorld',
        data() {
            return {
                results: []
            }
        },
        async created() {
            const r = await fetch('/api/values')
            this.results = await r.json()
        }
}
</script>