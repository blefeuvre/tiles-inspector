<template lang="pug">
#SelectSource
    select(@change="onSelectChange")
        option(v-for="(value) of this.sources" :value="value" :selected="value==selectedSource") {{value}}
</template>

<script>
    export default {
        name: 'SelectSource',
        props: {
        },
        data: () => ({
            selectedSource: undefined,
            sources: [],
        }),
        created() {
            window.SelectSource = this;
        },
        mounted() {
            this.initSources();
        },
        methods: {
            onSelectChange(event) {
                let select = event.target;
                this.selectedSource = select.options[select.selectedIndex].text;
                this.$emit('select', this.selectedSource);
            },
            evaluateXPath(aNode, aExpr) {
                const xpe = new XPathEvaluator();
                const nsResolver = xpe.createNSResolver(aNode.ownerDocument === null ? aNode.documentElement : aNode.ownerDocument.documentElement);
                const result = xpe.evaluate(aExpr, aNode, nsResolver, 0, null);
                const found = [];
                let res;
                while ((res = result.iterateNext())) {
                    found.push(res);
                }
                return found;
            },
            async searchDir(path) {
                let response = await fetch("http://localhost:8080"+path);
                response = await response.text();
                const htmlDocument = new DOMParser().parseFromString(response, "text/html");
                // search for metadata.json
                let results = this.evaluateXPath(htmlDocument, '//code[starts-with(., "(-")]/ancestor::tr[1]/td[@class="display-name"]/a');
                if (results.map((res) => res.textContent).filter((res) => res == "metadata.json").length) {
                    return path;
                }
                // iterate subdirs
                results = this.evaluateXPath(htmlDocument, '//code[starts-with(., "(d")]/ancestor::tr[1]/td[@class="display-name"]/a');
                const promises = results.map(
                    (res) => res.textContent
                ).filter((res) => res != "../").map(
                    (subdir) => this.searchDir(path+subdir)
                )
                return Promise.all(promises);
            },
            async initSources() {
                this.sources = (await this.searchDir("/")).flat().sort();
                this.selectedSource = this.sources[0];
                this.$emit('select', this.selectedSource);
            }
        },
    };
    </script>
    
    <style scoped>
    </style>
    