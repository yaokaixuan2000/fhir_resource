<script setup>
import {ref} from 'vue';
import {VueFlow} from '@vue-flow/core';
import {Background} from '@vue-flow/background';
import {Controls} from '@vue-flow/controls';
import {MiniMap} from '@vue-flow/minimap';
import Papa from 'papaparse';

const resourceName = ref([]); //target_resource
const allResources = ref({}); //reference_resource跟reference_path
const nodes = ref([]);
const edges = ref([]);

const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (file) {
    parseCSV(file);
  }
};

// Parse CSV file
const parseCSV = (file) => {
  Papa.parse(file, {
    header: true,
    complete: (result) => {
      const filteredData = result.data
          .filter(row => row.target_resource || row.reference_resource || row.reference_path)
          .map(row => ({
            target_resource: row.target_resource,
            target_resource_desc: row.target_resource_desc,
            reference_resource: row.reference_resource,
            reference_path: row.reference_path,
          }));

      const capitalizeFirstLetter = (string) => {
        return string.charAt(0).toUpperCase() + string.slice(1).toLowerCase();
      };

      //parent nodes名稱跟描述
      const resourceGroups = filteredData.reduce((acc, row) => {
        const key = capitalizeFirstLetter(row.target_resource);
        if (!acc[key]) {
          acc[key] = new Set([row.target_resource_desc]);
        } else {
          acc[key].add(row.target_resource_desc);
        }
        return acc;
      }, {});

      resourceName.value = Object.keys(resourceGroups).map(key => ({
        [key]: Array.from(resourceGroups[key]),
      }));

      const resourceSet = new Set(filteredData.map(row => capitalizeFirstLetter(row.reference_resource)));
      allResources.value = {};
      resourceSet.forEach(value => {
        allResources.value[value] = [];
      });

      assignResourcePaths(filteredData);
      createResourceNodes(resourceName.value);
    },
    error: (error) => {
      console.error('Error parsing CSV:', error);
    },
  });
};

const capitalizeFirstLetter = (string) => {
  return string.charAt(0).toUpperCase() + string.slice(1).toLowerCase();
}
const extractFirstPart = (path) => path.split('.')[0];

//小項分配
const assignResourcePaths = (filteredData) => {
  Object.keys(allResources.value).forEach(resource => {
    allResources.value[resource] = filteredData
        .filter(row => capitalizeFirstLetter(extractFirstPart(row.reference_path)) === resource)
        .map(row => ({
          referencePath: row.reference_path,
          targetResource: row.target_resource
        }));
  });
};

// parent nodes (target_resource)
const createResourceNodes = (resourceNames) => {
  const newNodes = resourceNames.map((name, index) => {
    const label = Object.keys(name)[0];
    const desc = name[label][0];
    return {
      id: `${index}`,
      data: {label, desc},
      position: {x: 50 + (index % 3) * 500, y: 110 + Math.floor(index / 3) * 300},
      style: {
        backgroundColor: 'rgba(255, 255, 255, 1)',
        fontSize: '20px',
        color: 'rgba(30, 144, 255, 1)',
        width: '430px',
        height: '100px',
        textAlign: 'left',
        padding: '10px',
        margin: '10px',
        borderRadius: '10px',
        borderColor: 'rgba(30, 144, 255, 1)',
      },
      sourcePosition: 'left',
      targetPosition: 'left',
    };
  });

  nodes.value = newNodes;
  handleChildNodes(resourceNames);
};

//child nodes (reference_path)
const handleChildNodes = (resourceNames) => {
  resourceNames.forEach((name, index) => {
    const resourceKey = Object.keys(name)[0];
    const paths = allResources.value[resourceKey]
        ? Array.from(new Set(allResources.value[resourceKey].map(item => item.referencePath)))
            .map(refPath => allResources.value[resourceKey].find(item => item.referencePath === refPath))
        : [];

    // console.log('paths:', paths);

    paths.forEach((path, pathIndex) => {
      const childHeight = 40;
      const margin = 20;
      const newHeight = 100 + paths.length * (childHeight + margin);
      nodes.value[index].style.height = `${newHeight}px`;
      const childNodesWidth = parseInt(nodes.value[index].style.width) - 20;

      const letterIndex = String.fromCharCode(97 + pathIndex);
      nodes.value.push({
        id: `${index}${letterIndex}`,
        data: {label: path.referencePath},
        position: {
          x: 10 + (index % 3) + 10,
          y: 10 + Math.floor(index / 3) + (pathIndex + 1) * (childHeight + margin),
        },
        style: {
          fontSize: '16px',
          color: 'rgba(50, 50, 50, 1)',
          width: `${childNodesWidth}px`,
          height: '40px',
          textAlign: 'left',
          padding: '10px',
          borderRadius: '8px',
          borderColor: 'rgba(255, 255, 255, 1)',
        },
        parentNode: String(index),
        extent: 'parent',
        sourcePosition: 'right',
        targetPosition: 'right',
      });

      const findTargetResourceID = nodes.value.find(node => node.data.label === capitalizeFirstLetter(path.targetResource));
      const findPathID = nodes.value.find(node => node.data.label === path.referencePath);
      // console.log('path:', path);
      // console.log('findPathID:', findPathID.id);
      // console.log('path.referencePath:', path.referencePath);
      // console.log('findTargetResourceID:', findTargetResourceID.id);
      // console.log('path.targetResource:', path.targetResource);

      //關聯線
      edges.value.push({
        id: `e${findPathID.id}-${findTargetResourceID.id}`,
        source: `${findPathID.id}`,//reference_resource
        target: `${findTargetResourceID.id}`,//target_resource
        animated: true,
        type: 'default',
        style: {stroke: 'rgba(255, 0, 0, 1)', strokeWidth: 2, zIndex: 10}
      });
    });


  });
  console.log('nodes:', nodes.value);
};

</script>

<template>
  <div>
    <div style="position: relative;">
      <h2>Upload CSV File</h2>
      <input type="file" @change="handleFileUpload"/>
    </div>
    <div class="vueflow-container">
      <VueFlow :nodes="nodes" :edges="edges" fit-view-on-init elevate-edges-on-select>
        <MiniMap/>
        <Controls/>
        <Background/>
      </VueFlow>
    </div>
  </div>
</template>


