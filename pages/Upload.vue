<template>

  <section class="container">

    <div>

<iframe src="https://ghbtns.com/github-btn.html?user=AlexisAnzieu&repo=zipackages-client&type=star&count=false" frameborder="0" scrolling="0" width="50px" height="20px"></iframe>
      <h1>
        {{title || "Drop your file"}}
      
      </h1>
      <h2>
        {{packageDescription || ""}}
      </h2>
      <no-ssr>
        <vue-dropzone ref="myVueDropzone" id="dropzone"  
          @vdropzone-file-added="added" 
          @vdropzone-sending="sending"
          @vdropzone-success="success" 
          @vdropzone-error="error" 
          :options="dropzoneOptions"
          v-show="showDropzone">
        </vue-dropzone>
      </no-ssr>

    </div>


<el-row v-if="dependenciesTable.length!=0"  > 
    <el-table id="listDependencies"
  v-if="dependenciesTable.length!=0"
    :data="dependenciesTable"
    stripe
    style="width: 100%">
    <el-table-column
      prop="package"
      label="Package">
    </el-table-column>
    <el-table-column
      prop="version"
      label="Version">
    </el-table-column>
  </el-table>
  <el-button v-if="!downloading" type="primary" @click="generate" round>Télécharger les librairies</el-button>
  <el-progress v-if="downloading" stroke-width=20 text-inside=true :percentage="percentage"></el-progress>
</el-row>

    <a href="/"> <el-button  v-if="downloaded" type="primary"  round>Come back to homepage</el-button> </a>

  </section>
</template>


<script>
// TODO:

var vue2Dropzone;
import ElementUI from "element-ui";
import axios from "axios";
import io from "socket.io-client";
import prettyBytes from "pretty-bytes";

if (process.browser) {
  vue2Dropzone = require("vue2-dropzone");
}

export default {
  data: function() {
    return {
      dropzoneOptions: {
        url: "https://apizipackages.now.sh/upload",
        dictDefaultMessage: "Drop your package.json here",
        headers: { "Cache-Control": null },
        autoProcessQueue: false
      },
      dependenciesTable: [],
      title: "",
      packageDescription: "",
      showDropzone: true,
      downloading: false,
      downloaded: false,
      socket: io("https://apizipackages.now.sh/"),
      totalSize: 0,
      percentage: 0
    };
  },
  components: {
    vueDropzone: vue2Dropzone
  },
  mounted() {
    this.socket.on("receiveSize", size => {
      this.totalSize = size;
      this.$message({
        message: `Packages estimed size : ${prettyBytes(size)}`,
        type: "success",
        duration: 4000
      });
    }),
      this.socket.on("generated", data => {
        if (data != 0) {
          this.$message({
            message: data,
            type: "error",
            duration: 6000
          });
        } else {
          this.percentage = 50;
          this.$message({
            message: "Files generated, zip in progress",
            type: "success",
            duration: 4000
          });
        }
      }),
      this.socket.on("zipped", filename => {
        if (filename === -1) {
          this.$message({
            message: "Issues during zip, sorry...",
            type: "success",
            duration: 4000
          });

          return;
        }

        this.percentage = 80;

        axios({
          url: `https://apizipackages.now.sh/download/${filename}`,
          method: "GET",
          responseType: "blob" // important
        }).then(response => {
          const url = window.URL.createObjectURL(new Blob([response.data]));
          const link = document.createElement("a");
          link.href = url;
          link.setAttribute("download", "node_modules.zip");
          document.body.appendChild(link);
          link.click();
          this.downloaded = true;
          this.title = "Thanks to use this tool !";
          this.dependenciesTable = [];
        });
      });
  },
  methods: {
    added: function(file) {
      var reader = new FileReader();
      reader.onload = e => {
        try {
          var readFile = JSON.parse(e.target.result);
        } catch (error) {
          this.$message({
            message: "Error, please drop a valid JSON format",
            type: "error",
            duration: 5000
          });

          this.$refs.myVueDropzone.removeAllFiles(); 

          throw "parsing failure";
        }

        let dependencies = {
          ...readFile.dependencies,
          ...readFile.devDependencies
        };
        this.title = readFile.name;
        this.packageDescription = readFile.description;
        this.showDropzone = false;

        for (let dependency in dependencies) {
          this.dependenciesTable.push({
            package: dependency,
            version: dependencies[dependency]
          });
        }
        this.socket.emit("getSize", JSON.stringify(this.dependenciesTable));
      };
      reader.readAsText(file);
    },
    generate: function() {
      this.$refs.myVueDropzone.processQueue();
    },
    sending: function() {
      this.downloading = true;

      this.$message({
        message: "Sending file...",
        type: "info",
        duration: 2000
      });
    },
    error: function() {
      this.downloading = false;

      this.$message({
        message: "Error, try again please",
        type: "error",
        duration: 5000
      });
    },
    success: function(file, filename) {
      this.socket.emit("createPackage", filename);

      this.$message({
        message: "Packages are being generated...",
        type: "success",
        duration: 4000
      });
    }
  }
};
</script>

<style>
h1 {
  font-size: 50px;
}
#listDependencies {
  text-align: left;
  margin-bottom: 20px;
}
</style>

