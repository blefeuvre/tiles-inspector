<template lang="pug">
#InfoMapBox
    | Zoom level :
    code {{ zoom }}
</template>
    
<script>
    export default {
        name: 'InfoMapBox',
        props: {
            zoom: Number,
        },
        data: () => ({}),
    
        created() {
            window.InfoMapBox = this;
        },
    };
    </script>
    
    <style scoped>
    code {
        font-weight: bold;
        margin-left: 10%;
    }
    </style>
    