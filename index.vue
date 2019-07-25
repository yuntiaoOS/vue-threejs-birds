<template>
  <div class="index">
    <script id="fragmentShaderPosition" type="x-shader/x-fragment">
			uniform float time;
			uniform float delta;
			void main()	{
				vec2 uv = gl_FragCoord.xy / resolution.xy;
				vec4 tmpPos = texture2D( texturePosition, uv );
				vec3 position = tmpPos.xyz;
				vec3 velocity = texture2D( textureVelocity, uv ).xyz;
				float phase = tmpPos.w;
				phase = mod( ( phase + delta +
					length( velocity.xz ) * delta * 3. +
					max( velocity.y, 0.0 ) * delta * 6. ), 62.83 );
				gl_FragColor = vec4( position + velocity * delta * 15. , phase );
			}
		</script>

		<!-- shader for bird's velocity -->
		<script id="fragmentShaderVelocity" type="x-shader/x-fragment">
			uniform float time;
			uniform float testing;
			uniform float delta; // about 0.016
			uniform float separationDistance; // 20
			uniform float alignmentDistance; // 40
			uniform float cohesionDistance; //
			uniform float freedomFactor;
			uniform vec3 predator;
			const float width = resolution.x;
			const float height = resolution.y;
			const float PI = 3.141592653589793;
			const float PI_2 = PI * 2.0;
			// const float VISION = PI * 0.55;
			float zoneRadius = 40.0;
			float zoneRadiusSquared = 1600.0;
			float separationThresh = 0.45;
			float alignmentThresh = 0.65;
			const float UPPER_BOUNDS = BOUNDS;
			const float LOWER_BOUNDS = -UPPER_BOUNDS;
			const float SPEED_LIMIT = 9.0;
			float rand( vec2 co ){
				return fract( sin( dot( co.xy, vec2(12.9898,78.233) ) ) * 43758.5453 );
			}
			void main() {
				zoneRadius = separationDistance + alignmentDistance + cohesionDistance;
				separationThresh = separationDistance / zoneRadius;
				alignmentThresh = ( separationDistance + alignmentDistance ) / zoneRadius;
				zoneRadiusSquared = zoneRadius * zoneRadius;
				vec2 uv = gl_FragCoord.xy / resolution.xy;
				vec3 birdPosition, birdVelocity;
				vec3 selfPosition = texture2D( texturePosition, uv ).xyz;
				vec3 selfVelocity = texture2D( textureVelocity, uv ).xyz;
				float dist;
				vec3 dir; // direction
				float distSquared;
				float separationSquared = separationDistance * separationDistance;
				float cohesionSquared = cohesionDistance * cohesionDistance;
				float f;
				float percent;
				vec3 velocity = selfVelocity;
				float limit = SPEED_LIMIT;
				dir = predator * UPPER_BOUNDS - selfPosition;
				dir.z = 0.;
				// dir.z *= 0.6;
				dist = length( dir );
				distSquared = dist * dist;
				float preyRadius = 150.0;
				float preyRadiusSq = preyRadius * preyRadius;
				// move birds away from predator
				if ( dist < preyRadius ) {
					f = ( distSquared / preyRadiusSq - 1.0 ) * delta * 100.;
					velocity += normalize( dir ) * f;
					limit += 5.0;
				}
				// if (testing == 0.0) {}
				// if ( rand( uv + time ) < freedomFactor ) {}
				// Attract flocks to the center
				vec3 central = vec3( 0., 0., 0. );
				dir = selfPosition - central;
				dist = length( dir );
				dir.y *= 2.5;
				velocity -= normalize( dir ) * delta * 5.;
				for ( float y = 0.0; y < height; y++ ) {
					for ( float x = 0.0; x < width; x++ ) {
						vec2 ref = vec2( x + 0.5, y + 0.5 ) / resolution.xy;
						birdPosition = texture2D( texturePosition, ref ).xyz;
						dir = birdPosition - selfPosition;
						dist = length( dir );
						if ( dist < 0.0001 ) continue;
						distSquared = dist * dist;
						if ( distSquared > zoneRadiusSquared ) continue;
						percent = distSquared / zoneRadiusSquared;
						if ( percent < separationThresh ) { // low
							// Separation - Move apart for comfort
							f = ( separationThresh / percent - 1.0 ) * delta;
							velocity -= normalize( dir ) * f;
						} else if ( percent < alignmentThresh ) { // high
							// Alignment - fly the same direction
							float threshDelta = alignmentThresh - separationThresh;
							float adjustedPercent = ( percent - separationThresh ) / threshDelta;
							birdVelocity = texture2D( textureVelocity, ref ).xyz;
							f = ( 0.5 - cos( adjustedPercent * PI_2 ) * 0.5 + 0.5 ) * delta;
							velocity += normalize( birdVelocity ) * f;
						} else {
							// Attraction / Cohesion - move closer
							float threshDelta = 1.0 - alignmentThresh;
							float adjustedPercent = ( percent - alignmentThresh ) / threshDelta;
							f = ( 0.5 - ( cos( adjustedPercent * PI_2 ) * -0.5 + 0.5 ) ) * delta;
							velocity += normalize( dir ) * f;
						}
					}
				}
				// this make tends to fly around than down or up
				// if (velocity.y > 0.) velocity.y *= (1. - 0.2 * delta);
				// Speed Limits
				if ( length( velocity ) > limit ) {
					velocity = normalize( velocity ) * limit;
				}
				gl_FragColor = vec4( velocity, 1.0 );
			}
		</script>

		<script type="x-shader/x-vertex" id="birdVS">
			attribute vec2 reference;
			attribute float birdVertex;
			attribute vec3 birdColor;
			uniform sampler2D texturePosition;
			uniform sampler2D textureVelocity;
			varying vec4 vColor;
			varying float z;
			uniform float time;
			void main() {
				vec4 tmpPos = texture2D( texturePosition, reference );
				vec3 pos = tmpPos.xyz;
				vec3 velocity = normalize(texture2D( textureVelocity, reference ).xyz);
				vec3 newPosition = position;
				if ( birdVertex == 4.0 || birdVertex == 7.0 ) {
					// flap wings
					newPosition.y = sin( tmpPos.w ) * 5.;
				}
				newPosition = mat3( modelMatrix ) * newPosition;
				velocity.z *= -1.;
				float xz = length( velocity.xz );
				float xyz = 1.;
				float x = sqrt( 1. - velocity.y * velocity.y );
				float cosry = velocity.x / xz;
				float sinry = velocity.z / xz;
				float cosrz = x / xyz;
				float sinrz = velocity.y / xyz;
				mat3 maty =  mat3(
					cosry, 0, -sinry,
					0    , 1, 0     ,
					sinry, 0, cosry
				);
				mat3 matz =  mat3(
					cosrz , sinrz, 0,
					-sinrz, cosrz, 0,
					0     , 0    , 1
				);
				newPosition =  maty * matz * newPosition;
				newPosition += pos;
				z = newPosition.z;
				vColor = vec4( birdColor, 1.0 );
				gl_Position = projectionMatrix *  viewMatrix  * vec4( newPosition, 1.0 );
			}
		</script>

		<!-- bird geometry shader -->
<!--    void main() {-->
<!--				// Fake colors for now-->
<!--				float z2 = 0.2 + ( 1000. - z ) / 1000. * vColor.x;-->
<!--				gl_FragColor = vec4( z2, z2, z2, 1. );-->

<!--			}-->
		<script type="x-shader/x-fragment" id="birdFS">
			varying vec4 vColor;
			varying float z;
			uniform vec3 color;
			void main() {
				// Fake colors for now
				float rr = 0.2 + ( 1000. - z ) / 1000. * vColor.x;
        float gg = 0.2 + ( 1000. - z ) / 1000. * vColor.y;
        float bb = 0.2 + ( 1000. - z ) / 1000. * vColor.z;
        gl_FragColor = vec4( rr, gg, bb, 1. );

			}
		</script>
    <div id="container">
    </div>
  </div>


</template>




<script>
import * as THREE from 'three'
import { GUI } from '@/libs/dat.gui.module.js'
import { GPUComputationRenderer } from '@/libs/GPUComputationRenderer.js'

export default {
  name: 'ThreeTest',
  data() {

    return {
      WIDTH : 32,
      BIRDS : 32*32,

      camera: null,
      scene: null,
      renderer: null,
      BirdGeometry: Object.create(null),
      windowHalfX: window.innerWidth / 2,
      windowHalfY: window.innerHeight / 2,
       container: null,
      BOUNDS : 800,
      BOUNDS_HALF : 800 / 2,
      mouseY: 0,
      mouseX: 0,
      last: window.performance.now(),
      gpuCompute: null,
      velocityVariable: {},
      positionVariable: {},
      positionUniforms: {},
      velocityUniforms: {},
      birdUniforms: null,
    }
  },

  mounted() {

    this.BirdGeometry = this.BirdGeometryF()

    this.init()
    this.animate()
  },
  methods: {

    init: function() {
       this.container = document.getElementById('container')

      this.camera = new THREE.PerspectiveCamera( 75, window.innerWidth / window.innerHeight, 1, 3000 );
				this.camera.position.z = 350;
				this.scene = new THREE.Scene();
				this.scene.background = new THREE.Color( 0xffffff );
				this.scene.fog = new THREE.Fog( 0xffffff, 100, 1000 );
				this.renderer = new THREE.WebGLRenderer();
				this.renderer.setPixelRatio( window.devicePixelRatio );
				this.renderer.setSize( window.innerWidth, window.innerHeight );
				this.container.appendChild( this.renderer.domElement );
				this.initComputeRenderer();

				document.addEventListener( 'mousemove', this.onDocumentMouseMove, false );
				document.addEventListener( 'touchstart', this.onDocumentTouchStart, false );
				document.addEventListener( 'touchmove', this.onDocumentTouchMove, false );
				//
				window.addEventListener( 'resize', this.onWindowResize, false );
				var gui = new GUI();
				var effectController = {
					separation: 20.0,
					alignment: 20.0,
					cohesion: 20.0,
					freedom: 0.75
				};

				this.valuesChanger();
				gui.add( effectController, "separation", 0.0, 100.0, 1.0 ).onChange( this.valuesChanger );
				gui.add( effectController, "alignment", 0.0, 100, 0.001 ).onChange( this.valuesChanger );
				gui.add( effectController, "cohesion", 0.0, 100, 0.025 ).onChange( this.valuesChanger );
				gui.close();
				this.initBirds();
    },
   valuesChanger  () {

       var effectController = {
					separation: 20.0,
					alignment: 20.0,
					cohesion: 20.0,
					freedom: 0.75
				}
					this.velocityUniforms[ "separationDistance" ].value = effectController.separation;
					this.velocityUniforms[ "alignmentDistance" ].value = effectController.alignment;
					this.velocityUniforms[ "cohesionDistance" ].value = effectController.cohesion;
					this.velocityUniforms[ "freedomFactor" ].value = effectController.freedom;
				},
    BirdGeometryF(){

        var birdG = new THREE.BufferGeometry()

				var triangles = this.BIRDS * 3;
				var points = triangles * 3;

				var vertices = new THREE.BufferAttribute( new Float32Array( points * 3 ), 3 );
				var birdColors = new THREE.BufferAttribute( new Float32Array( points * 3 ), 3 );
				var references = new THREE.BufferAttribute( new Float32Array( points * 2 ), 2 );
				var birdVertex = new THREE.BufferAttribute( new Float32Array( points ), 1 );
				birdG.addAttribute( 'position', vertices );
				birdG.addAttribute( 'birdColor', birdColors );
				birdG.addAttribute( 'reference', references );
				birdG.addAttribute( 'birdVertex', birdVertex );

				// this.addAttribute( 'normal', new Float32Array( points * 3 ), 3 );
				var v = 0;
				function verts_push() {
					for ( var i = 0; i < arguments.length; i ++ ) {
						vertices.array[ v ++ ] = arguments[ i ];
					}
				}
				var wingsSpan = 20;
				for ( var f = 0; f < this.BIRDS; f ++ ) {
					// Body
					verts_push(
						0, - 0, - 20,
						0, 4, - 20,
						0, 0, 30
					);
					// Left Wing
					verts_push(
						0, 0, - 15,
						- wingsSpan, 0, 0,
						0, 0, 15
					);
					// Right Wing
					verts_push(
						0, 0, 15,
						wingsSpan, 0, 0,
						0, 0, - 15
					);

				}

				const colorCache = {}

				for ( var v = 0; v < triangles * 3; v ++ ) {
					var i = ~ ~ ( v / 3 );
					var x = ( i % this.WIDTH ) / this.WIDTH;
					var y = ~ ~ ( i / this.WIDTH ) / this.WIDTH;
					// var c = new THREE.Color(
					// 	0x444444 +
					// 	~ ~ ( v / 9 ) /this.BIRDS * 0x666666
					// );


          // const order =  ~~(v / 9) / this.BIRDS

          const order = Math.floor((v / 9) / this.BIRDS)
          const key = order.toString()
          const gradient = 'varianceGradient'.indexOf('Gradient') !== -1

          let c

          if (!gradient && colorCache[key]) {
            c = colorCache[key]
          } else {

            c = this.getNewCol(order)
          }

          if (!gradient && !colorCache[key]) {
            colorCache[key] = c
          }



					birdColors.array[ v * 3 + 0 ] = c.r;
					birdColors.array[ v * 3 + 1 ] = c.g;
					birdColors.array[ v * 3 + 2 ] = c.b;
					references.array[ v * 2 ] = x;
					references.array[ v * 2 + 1 ] = y;
					birdVertex.array[ v ] = v % 9;
				}
				birdG.scale( 0.2, 0.2, 0.2 );
				return  birdG
			},
    getNewCol(order) {

    const color1 =  0xe11443
    const color2 =  0x17c0e6
    const c1 = new THREE.Color(color1)
    const c2 = new THREE.Color(color2)
    const gradient = 'varianceGradient'.indexOf('Gradient') !== -1
    let c, dist
    if (gradient) {
      // each vertex has a different color
      dist = Math.random()

    } else {
      // each vertex has the same color
      dist = order
    }

    if ('varianceGradient'.indexOf('variance') === 0) {

       const r2 = THREE.Math.clamp((c1.r + Math.random() * c2.r),0,1)
      const g2 = THREE.Math.clamp((c1.g + Math.random() * c2.g),0,1)
      const b2 = THREE.Math.clamp((c1.b + Math.random() * c2.b),0,1)
      // const r2 = (c1.r + Math.random() * c2.r).clamp(0,1)
      // const g2 = (c1.g + Math.random() * c2.g).clamp(0,1)
      // const b2 = (c1.b + Math.random() * c2.b).clamp(0,1)

      c = new THREE.Color(r2, g2, b2)

    } else if ('varianceGradient'.indexOf('mix') === 0) {
      // Naive color arithmetic
      c = new THREE.Color(color1 + dist * color2)
    } else {

      // Linear interpolation
      c = c1.lerp(c2, dist)
    }
    return c
  },
    initComputeRenderer() {
				this.gpuCompute = new GPUComputationRenderer( this.WIDTH, this.WIDTH, this.renderer );
				var dtPosition = this.gpuCompute.createTexture();
				var dtVelocity = this.gpuCompute.createTexture();
				this.fillPositionTexture( dtPosition );
				this.fillVelocityTexture( dtVelocity );
				this.velocityVariable = this.gpuCompute.addVariable( "textureVelocity", document.getElementById( 'fragmentShaderVelocity' ).textContent, dtVelocity );
				this.positionVariable = this.gpuCompute.addVariable( "texturePosition", document.getElementById( 'fragmentShaderPosition' ).textContent, dtPosition );
				this.gpuCompute.setVariableDependencies( this.velocityVariable, [ this.positionVariable, this.velocityVariable ] );
				this.gpuCompute.setVariableDependencies( this.positionVariable, [ this.positionVariable, this.velocityVariable ] );
				this.positionUniforms = this.positionVariable.material.uniforms;
				this.velocityUniforms = this.velocityVariable.material.uniforms;
				this.positionUniforms[ "time" ] = { value: 0.0 };
				this.positionUniforms[ "delta" ] = { value: 0.0 };
				this.velocityUniforms[ "time" ] = { value: 1.0 };
				this.velocityUniforms[ "delta" ] = { value: 0.0 };
				this.velocityUniforms[ "testing" ] = { value: 1.0 };
				this.velocityUniforms[ "separationDistance" ] = { value: 1.0 };
				this.velocityUniforms[ "alignmentDistance" ] = { value: 1.0 };
				this.velocityUniforms[ "cohesionDistance" ] = { value: 1.0 };
				this.velocityUniforms[ "freedomFactor" ] = { value: 1.0 };
				this.velocityUniforms[ "predator" ] = { value: new THREE.Vector3() };
				this.velocityVariable.material.defines.BOUNDS = this.BOUNDS.toFixed( 2 );
				this.velocityVariable.wrapS = THREE.RepeatWrapping;
				this.velocityVariable.wrapT = THREE.RepeatWrapping;
				this.positionVariable.wrapS = THREE.RepeatWrapping;
				this.positionVariable.wrapT = THREE.RepeatWrapping;
				var error = this.gpuCompute.init();
				if ( error !== null ) {
				    console.error( error );
				}
			},
    initBirds() {
				var geometry = this.BirdGeometryF()
				// For Vertex and Fragment
				this.birdUniforms = {
					"color": { value: new THREE.Color( 0xff2200 ) },
					"texturePosition": { value: null },
					"textureVelocity": { value: null },
					"time": { value: 1.0 },
					"delta": { value: 0.0 }
				};
				// THREE.ShaderMaterial
				var material = new THREE.ShaderMaterial( {
					uniforms: this.birdUniforms,
					vertexShader: document.getElementById( 'birdVS' ).textContent,
					fragmentShader: document.getElementById( 'birdFS' ).textContent,
					side: THREE.DoubleSide
				} );
				var birdMesh = new THREE.Mesh( geometry, material );
				birdMesh.rotation.y = Math.PI / 2;
				birdMesh.matrixAutoUpdate = false;
				birdMesh.updateMatrix();
				this.scene.add( birdMesh );
			},
    fillPositionTexture( texture ) {
				var theArray = texture.image.data;
				for ( var k = 0, kl = theArray.length; k < kl; k += 4 ) {
					var x = Math.random() * this.BOUNDS - this.BOUNDS_HALF;
					var y = Math.random() * this.BOUNDS - this.BOUNDS_HALF;
					var z = Math.random() * this.BOUNDS - this.BOUNDS_HALF;
					theArray[ k + 0 ] = x;
					theArray[ k + 1 ] = y;
					theArray[ k + 2 ] = z;
					theArray[ k + 3 ] = 1;
				}
			},
    fillVelocityTexture( texture ) {
				var theArray = texture.image.data;
				for ( var k = 0, kl = theArray.length; k < kl; k += 4 ) {
					var x = Math.random() - 0.5;
					var y = Math.random() - 0.5;
					var z = Math.random() - 0.5;
					theArray[ k + 0 ] = x * 10;
					theArray[ k + 1 ] = y * 10;
					theArray[ k + 2 ] = z * 10;
					theArray[ k + 3 ] = 1;
				}
			},
     onWindowResize() {
				this.windowHalfX = window.innerWidth / 2;
				this.windowHalfY = window.innerHeight / 2;
				this.camera.aspect = window.innerWidth / window.innerHeight;
				this.camera.updateProjectionMatrix();
				this.renderer.setSize( window.innerWidth, window.innerHeight );
			},
			 onDocumentMouseMove( event ) {
				this.mouseX = event.clientX - this.windowHalfX - 200;
				this.mouseY = event.clientY - this.windowHalfY - 100;
			},
			 onDocumentTouchStart( event ) {
				if ( event.touches.length === 1 ) {
					event.preventDefault();
					this.mouseX = event.touches[ 0 ].pageX - this.windowHalfX;
					this.mouseY = event.touches[ 0 ].pageY - this.windowHalfY;
				}
			},
			 onDocumentTouchMove( event ) {
				if ( event.touches.length === 1 ) {
					event.preventDefault();
					this.mouseX = event.touches[ 0 ].pageX - this.windowHalfX;
					this.mouseY = event.touches[ 0 ].pageY - this.windowHalfY;
				}
			},
			//
			 animate() {
				requestAnimationFrame( this.animate );
				this.render();

			},
    render() {
				var now = window.performance.now();
				var delta = ( now - this.last ) / 1000;
				if ( delta > 1 ) delta = 1; // safety cap on large deltas
				this.last = now;
				this.positionUniforms[ "time" ].value = now;
				this.positionUniforms[ "delta" ].value = delta;
				this.velocityUniforms[ "time" ].value = now;
				this.velocityUniforms[ "delta" ].value = delta;
				this.birdUniforms[ "time" ].value = now;
				this.birdUniforms[ "delta" ].value = delta;
				this.velocityUniforms[ "predator" ].value.set( 0.5 * this.mouseX / this.windowHalfX, - 0.5 * this.mouseY / this.windowHalfY, 0 );
				this.mouseX = 10000;
				this.mouseY = 10000;
				this.gpuCompute.compute();
				this.birdUniforms[ "texturePosition" ].value = this.gpuCompute.getCurrentRenderTarget( this.positionVariable ).texture;
				this.birdUniforms[ "textureVelocity" ].value = this.gpuCompute.getCurrentRenderTarget( this.velocityVariable ).texture;
				this.renderer.render( this.scene, this.camera );
			}


  }
}
</script>
<style scoped>
  .index {
  width: 100%;
  position: absolute;
  top: 0;
  bottom: 0;
  left: 0;
  text-align: center;
  background-color: #141a48;
  background-repeat: no-repeat;
  background-size: cover;
  overflow: hidden;
}
  #container {
    position: absolute;
    width: 100%;
    top: 0;
    bottom: 0;
    overflow: hidden;

  }
</style>
