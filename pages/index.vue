<template>
  <div class="container">
    <input type="file" @change="onFileChange" accept="image/*">
    <h1>{{license}}</h1>
  </div>
</template>

<script>

    export default {
        data(){
            return {
                image: '',
                license: 'License will appear here',
                apiKey: "AIzaSyCX7hdA5Sj0TeyQuqL-ZyUewyt9GJ1mvZ0"
            }
        },
        methods: {
            onFileChange(e) {
                var files = e.target.files || e.dataTransfer.files;
                if (!files.length)
                    return;
                this.createImage(files[0]);
            },
            createImage(file) {
                console.log('loading...........');
                this.license= "Loading........"
                var image = new Image();
                var reader = new FileReader();
                var vm = this;

                reader.onload = (e) => {
                    vm.image = e.target.result;
                    this.$axios.post("https://vision.googleapis.com/v1/images:annotate", {
                        "requests": [
                            {
                                "image": {
                                    "content": this.image.split(',')[1]
                                },
                                "features": [
                                    {
                                        "type": "TEXT_DETECTION"
                                    }
                                ]
                            }
                        ]
                    }, {
                        params: {
                            key: this.apiKey
                        }
                    }).then(res => {
                        console.log(res.data)
                        let responses= res.data.responses
                        if(!responses || responses.length <= 0)
                            return false;

                        let textAnnotations= responses[0].textAnnotations

                        if(!textAnnotations || textAnnotations.length <= 0)
                            return false;

                        let licenseNo= false
                        Object.keys(textAnnotations).forEach(function (key) {
                            if(licenseNo)
                                return;
                            key= parseInt(key)
                            //check patterns
                            // console.log("Key " + key + " ,Description " + textAnnotations[key].description)
                            if(textAnnotations[key].description == "رقم") {
                                console.log(key)
                                if(textAnnotations.length > key + 1 && (textAnnotations[key + 1].description.startsWith("الرخصة"))){
                                    if(textAnnotations.length > key + 2 && textAnnotations[key + 2].description.match(/^\d/)){
                                        console.log("found it "+ textAnnotations[key + 2].description)
                                        licenseNo= textAnnotations[key + 2].description
                                    }
                                }
                            }
                            if(textAnnotations[key].description.toLowerCase() == "license" || textAnnotations[key].description.toLowerCase() == "licence"){
                                if(textAnnotations.length > key + 1 && (textAnnotations[key + 1].description.toLowerCase().startsWith("no"))){
                                    // console.log("2")
                                    if(textAnnotations.length > key + 2 && textAnnotations[key + 2].description.match(/^\d/)){
                                        console.log("found it "+ textAnnotations[key + 2].description)
                                        licenseNo= textAnnotations[key + 2].description
                                    }
                                }
                            }
                        })
                        return licenseNo
                    }).then(res => {
                        this.license= res
                        console.log("Finished Loading")
                    }).catch(err => {
                        console.log('error')
                        console.log(err)
                        this.license= "Failed"
                    })
                };
                reader.readAsDataURL(file);
            },
            removeImage: function (e) {
                this.image = '';
            }
        }
    }
</script>

<style>
</style>
