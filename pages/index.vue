<template>
  <div class="container">
    <canvas id="the-canvas" style="display: none"></canvas>
    <input id="fileInput" type="file" @change="onFileChange" accept="image/*,.pdf">
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
                var reader = new FileReader();
                var vm = this;
                let fileInput= document.getElementById('fileInput').files[0]

                reader.onload = (e) => {
                    vm.image = e.target.result;
                    if (fileInput.type.match('image.*')) {
                        console.log("is an image");
                        this.checkLicense()
                    }else if(fileInput.type.match('.*pdf')){
                        console.log("is a pdf");
                        this.pdfToImg()
                    }else{
                        this.license= "Wrong file type"
                    }
                };
                reader.readAsDataURL(file);
            },
            removeImage: function (e) {
                this.image = '';
            },
            pdfToImg(){
                // let pdfjsLib = window['pdfjs-dist/build/pdf'];
                pdfjsLib.GlobalWorkerOptions.workerSrc = '//mozilla.github.io/pdf.js/build/pdf.worker.js';

                let loadingTask = pdfjsLib.getDocument({data: atob(this.image.split(',')[1])});
                loadingTask.promise.then((pdf) => {
                    console.log('PDF loaded');

                    // Fetch the first page
                    let pageNumber = 1;
                    pdf.getPage(pageNumber).then(page => {
                        console.log('Page loaded');

                        var scale = 1.5;
                        var viewport = page.getViewport({scale: scale});

                        // Prepare canvas using PDF page dimensions
                        var canvas = document.getElementById('the-canvas');
                        var context = canvas.getContext('2d');
                        canvas.height = viewport.height;
                        canvas.width = viewport.width;

                        // Render PDF page into canvas context
                        var renderContext = {
                            canvasContext: context,
                            viewport: viewport
                        };
                        var renderTask = page.render(renderContext);
                        renderTask.promise.then(() => {
                            console.log('Page rendered');
                            // console.log(canvas.toDataURL())
                            this.image= canvas.toDataURL()
                            this.checkLicense()
                        });
                    });
                }, (reason) => {
                    // PDF loading error
                    console.error(reason);
                });
            },
            checkLicense(){
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
                            ],
                            "imageContext": {
                                "languageHints": ["en"]
                            }
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
                                if(textAnnotations.length > key + 2 && textAnnotations[key + 2].description.match(/\d/)){
                                    console.log("found it "+ textAnnotations[key + 2].description)
                                    licenseNo= textAnnotations[key + 2].description
                                }
                            }
                        }
                        if(textAnnotations[key].description.toLowerCase() == "license" || textAnnotations[key].description.toLowerCase() == "licence"){
                            console.log(key)
                            if(textAnnotations.length > key + 1 && (textAnnotations[key + 1].description.toLowerCase().startsWith("no"))){
                                if(textAnnotations.length > key + 2 && textAnnotations[key + 2].description.match(/\d/)){
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
            }
        }
    }
</script>

<style>
</style>
