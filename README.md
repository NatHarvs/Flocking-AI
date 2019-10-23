# Flocking-AI
Flocking AI

In this project, I used the Coherent, Alignment and Avoidance behaviours to simulate flocking with a group of agents. I was given skeleton code to work from that provided a single agent being rendered and function names given.

DrawableGameObject.h

#pragma once

#include <d3d11_1.h>
#include <d3dcompiler.h>
#include <directxcolors.h>
#include <DirectXCollision.h>
#include "DDSTextureLoader.h"
#include "resource.h"
#include <iostream>
#include "structures.h"
#include <ctime>

/*
 movement
 a current direction vector
 speed is constant
 for now, new directions are instant (no rotation periods)
*/

using namespace DirectX;



struct SimpleVertex
{
	XMFLOAT3 Pos;
	XMFLOAT3 Normal;
	XMFLOAT2 TexCoord;
};

class DrawableGameObject
{
public:
	DrawableGameObject();
	~DrawableGameObject();

	HRESULT								initMesh(ID3D11Device* pd3dDevice, ID3D11DeviceContext* pContext);
	void								update(float t, const vecDrawables& drawList, const unsigned int index);
	void								draw(ID3D11DeviceContext* pContext);
	ID3D11Buffer*						getVertexBuffer() { return m_pVertexBuffer; }
	ID3D11Buffer*						getIndexBuffer() { return m_pIndexBuffer; }
	ID3D11ShaderResourceView**			getTextureResourceView() { return &m_pTextureResourceView; 	}
	XMMATRIX*							getTransform() { return &m_World; }
	ID3D11SamplerState**				getTextureSamplerState() { return &m_pSamplerLinear; }
	MaterialPropertiesConstantBuffer	getMaterial() { return m_material;}

	void								setPosition(XMFLOAT3 position);
	XMFLOAT3*							getPosition() { return &m_position; }
	XMFLOAT3*							getDirection() { return &m_direction; }

	void								checkIsOnScreenAndFix(const XMMATRIX&  view, const XMMATRIX&  proj);

protected:
	void								setDirection(XMFLOAT3 direction); 

	void								nearbyDrawables(const vecDrawables& drawList, const unsigned int index);
	void								furtherDrawables(const vecDrawables& drawList, const unsigned int index);
	XMFLOAT3							calculateSeparationVector(const vecDrawables& drawList);
	XMFLOAT3							calculateAlignmentVector(const vecDrawables& drawList);
	XMFLOAT3							calculateCohesionVector(const vecDrawables& drawList);
	void								createRandomDirection();

	//	Helpful math functions
	XMFLOAT3							addFloat3(XMFLOAT3& f1, XMFLOAT3& f2);
	XMFLOAT3							subtractFloat3(XMFLOAT3& f1, XMFLOAT3& f2);
	XMFLOAT3							normaliseFloat3(XMFLOAT3& f1);
	float								magnitudeFloat3(XMFLOAT3& f1);
	XMFLOAT3							multiplyFloat3(XMFLOAT3& f1, const float scalar);
	XMFLOAT3							divideFloat3(XMFLOAT3& f1, const float scalar);

private:
	
	XMMATRIX							m_World;
	ID3D11Buffer*						m_pVertexBuffer;
	ID3D11Buffer*						m_pIndexBuffer;
	ID3D11ShaderResourceView*			m_pTextureResourceView;
	ID3D11SamplerState *				m_pSamplerLinear;
	MaterialPropertiesConstantBuffer	m_material;
	XMFLOAT3							m_position;

	XMFLOAT3							m_direction;
	unsigned int*						m_nearbyDrawables;
	unsigned int*						m_furtherDrawables;


};

DrawableGameObject.cpp

#include "DrawableGameObject.h"
#include <stdlib.h>
#include <ctime>

using namespace std;
using namespace DirectX;

#define NUM_VERTICES		36 // do not change!

#define NEARBY_DISTANCE		10.0f		//Use for Avoidance

#define FURTHER_DISTANCE	20.0f		//Use for Coherence

#define OBJECT_SPEED		50.0f

DrawableGameObject::DrawableGameObject()
{
	m_pVertexBuffer = nullptr;
	m_pIndexBuffer = nullptr;
	m_pTextureResourceView = nullptr;
	m_pSamplerLinear = nullptr;

	// Initialize the world matrix
	m_World = XMMatrixIdentity();

	m_nearbyDrawables = nullptr;
	m_furtherDrawables = nullptr;
	m_position = XMFLOAT3(0, 0, 0);

	Sleep(30);

	createRandomDirection();
}

void DrawableGameObject::createRandomDirection()
{
	//Randomly generate a direction for each agent
	float directionx = 2.0 * (rand() / (float)RAND_MAX) -1;
	float directiony = 2.0 * (rand() / (float)RAND_MAX) -1;

	m_direction = XMFLOAT3(directionx, directiony, 0);
	m_direction = normaliseFloat3(m_direction);
	setDirection(m_direction);
}

void DrawableGameObject::setDirection(XMFLOAT3 direction)
{
	XMVECTOR v = XMLoadFloat3(&direction);
	v = XMVector3Normalize(v);
	XMStoreFloat3(&m_direction, v);
}

DrawableGameObject::~DrawableGameObject()
{
	if( m_pVertexBuffer ) 
		m_pVertexBuffer->Release();
	
	if( m_pIndexBuffer )
		m_pIndexBuffer->Release();	

	if (m_pTextureResourceView)
		m_pTextureResourceView->Release();

	if (m_pSamplerLinear)
		m_pSamplerLinear->Release();

	if (m_nearbyDrawables != nullptr) {
		delete [] m_nearbyDrawables;
	}

	if (m_furtherDrawables != nullptr) {
		delete[] m_furtherDrawables;
	}
}

HRESULT DrawableGameObject::initMesh(ID3D11Device* pd3dDevice, ID3D11DeviceContext* pContext)
{
	// Create vertex buffer
	SimpleVertex vertices[] =
	{
		{ XMFLOAT3(-1.0f, 1.0f, -1.0f), XMFLOAT3(0.0f, 1.0f, 0.0f), XMFLOAT2(1.0f, 0.0f) },
		{ XMFLOAT3(1.0f, 1.0f, -1.0f), XMFLOAT3(0.0f, 1.0f, 0.0f), XMFLOAT2(0.0f, 0.0f) },
		{ XMFLOAT3(1.0f, 1.0f, 1.0f), XMFLOAT3(0.0f, 1.0f, 0.0f), XMFLOAT2(0.0f, 1.0f) },
		{ XMFLOAT3(-1.0f, 1.0f, 1.0f), XMFLOAT3(0.0f, 1.0f, 0.0f), XMFLOAT2(1.0f, 1.0f) },

		{ XMFLOAT3(-1.0f, -1.0f, -1.0f), XMFLOAT3(0.0f, -1.0f, 0.0f), XMFLOAT2(0.0f, 0.0f) },
		{ XMFLOAT3(1.0f, -1.0f, -1.0f), XMFLOAT3(0.0f, -1.0f, 0.0f), XMFLOAT2(1.0f, 0.0f) },
		{ XMFLOAT3(1.0f, -1.0f, 1.0f), XMFLOAT3(0.0f, -1.0f, 0.0f), XMFLOAT2(1.0f, 1.0f) },
		{ XMFLOAT3(-1.0f, -1.0f, 1.0f), XMFLOAT3(0.0f, -1.0f, 0.0f), XMFLOAT2(0.0f, 1.0f) },

		{ XMFLOAT3(-1.0f, -1.0f, 1.0f), XMFLOAT3(-1.0f, 0.0f, 0.0f), XMFLOAT2(0.0f, 1.0f) },
		{ XMFLOAT3(-1.0f, -1.0f, -1.0f), XMFLOAT3(-1.0f, 0.0f, 0.0f), XMFLOAT2(1.0f, 1.0f) },
		{ XMFLOAT3(-1.0f, 1.0f, -1.0f), XMFLOAT3(-1.0f, 0.0f, 0.0f), XMFLOAT2(1.0f, 0.0f) },
		{ XMFLOAT3(-1.0f, 1.0f, 1.0f), XMFLOAT3(-1.0f, 0.0f, 0.0f), XMFLOAT2(0.0f, 0.0f) },

		{ XMFLOAT3(1.0f, -1.0f, 1.0f), XMFLOAT3(1.0f, 0.0f, 0.0f), XMFLOAT2(1.0f, 1.0f) },
		{ XMFLOAT3(1.0f, -1.0f, -1.0f), XMFLOAT3(1.0f, 0.0f, 0.0f), XMFLOAT2(0.0f, 1.0f) },
		{ XMFLOAT3(1.0f, 1.0f, -1.0f), XMFLOAT3(1.0f, 0.0f, 0.0f), XMFLOAT2(0.0f, 0.0f) },
		{ XMFLOAT3(1.0f, 1.0f, 1.0f), XMFLOAT3(1.0f, 0.0f, 0.0f), XMFLOAT2(1.0f, 0.0f) },

		{ XMFLOAT3(-1.0f, -1.0f, -1.0f), XMFLOAT3(0.0f, 0.0f, -1.0f), XMFLOAT2(0.0f, 1.0f) },
		{ XMFLOAT3(1.0f, -1.0f, -1.0f), XMFLOAT3(0.0f, 0.0f, -1.0f) , XMFLOAT2(1.0f, 1.0f) },
		{ XMFLOAT3(1.0f, 1.0f, -1.0f), XMFLOAT3(0.0f, 0.0f, -1.0f), XMFLOAT2(1.0f, 0.0f) },
		{ XMFLOAT3(-1.0f, 1.0f, -1.0f), XMFLOAT3(0.0f, 0.0f, -1.0f), XMFLOAT2(0.0f, 0.0f) },

		{ XMFLOAT3(-1.0f, -1.0f, 1.0f), XMFLOAT3(0.0f, 0.0f, 1.0f), XMFLOAT2(1.0f, 1.0f) },
		{ XMFLOAT3(1.0f, -1.0f, 1.0f), XMFLOAT3(0.0f, 0.0f, 1.0f), XMFLOAT2(0.0f, 1.0f) },
		{ XMFLOAT3(1.0f, 1.0f, 1.0f), XMFLOAT3(0.0f, 0.0f, 1.0f), XMFLOAT2(0.0f, 0.0f) },
		{ XMFLOAT3(-1.0f, 1.0f, 1.0f), XMFLOAT3(0.0f, 0.0f, 1.0f), XMFLOAT2(1.0f, 0.0f) },
	};

	D3D11_BUFFER_DESC bd = {};
	bd.Usage = D3D11_USAGE_DEFAULT;
	bd.ByteWidth = sizeof(SimpleVertex) * 24;
	bd.BindFlags = D3D11_BIND_VERTEX_BUFFER;
	bd.CPUAccessFlags = 0;

	D3D11_SUBRESOURCE_DATA InitData = {};
	InitData.pSysMem = vertices;
	HRESULT hr = pd3dDevice->CreateBuffer(&bd, &InitData, &m_pVertexBuffer);
	if (FAILED(hr))
		return hr;

	// Set vertex buffer
	UINT stride = sizeof(SimpleVertex);
	UINT offset = 0;
	pContext->IASetVertexBuffers(0, 1, &m_pVertexBuffer, &stride, &offset);

	// Create index buffer
	WORD indices[] =
	{
		3,1,0,
		2,1,3,

		6,4,5,
		7,4,6,

		11,9,8,
		10,9,11,

		14,12,13,
		15,12,14,

		19,17,16,
		18,17,19,

		22,20,21,
		23,20,22
	};

	bd.Usage = D3D11_USAGE_DEFAULT;
	bd.ByteWidth = sizeof(WORD) * NUM_VERTICES;        // 36 vertices needed for 12 triangles in a triangle list
	bd.BindFlags = D3D11_BIND_INDEX_BUFFER;
	bd.CPUAccessFlags = 0;
	InitData.pSysMem = indices;
	hr = pd3dDevice->CreateBuffer(&bd, &InitData, &m_pIndexBuffer);
	if (FAILED(hr))
		return hr;

	// Set index buffer
	pContext->IASetIndexBuffer(m_pIndexBuffer, DXGI_FORMAT_R16_UINT, 0);

	// Set primitive topology
	pContext->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);

	// load and setup textures
	hr = CreateDDSTextureFromFile(pd3dDevice, L"Resources\\stone.dds", nullptr, &m_pTextureResourceView);
	if (FAILED(hr))
		return hr;

	D3D11_SAMPLER_DESC sampDesc;
	ZeroMemory(&sampDesc, sizeof(sampDesc));
	sampDesc.Filter = D3D11_FILTER_ANISOTROPIC;
	sampDesc.AddressU = D3D11_TEXTURE_ADDRESS_WRAP;
	sampDesc.AddressV = D3D11_TEXTURE_ADDRESS_WRAP;
	sampDesc.AddressW = D3D11_TEXTURE_ADDRESS_WRAP;
	sampDesc.ComparisonFunc = D3D11_COMPARISON_NEVER;
	sampDesc.MinLOD = 0;
	sampDesc.MaxLOD = D3D11_FLOAT32_MAX;
	hr = pd3dDevice->CreateSamplerState(&sampDesc, &m_pSamplerLinear);

	m_material.Material.Diffuse = XMFLOAT4(1.0f, 1.0f, 1.0f, 1.0f);
	m_material.Material.Specular = XMFLOAT4(1.0f, 0.2f, 0.2f, 1.0f);
	m_material.Material.SpecularPower = 32.0f;
	m_material.Material.UseTexture = true;

	return hr;
}

void DrawableGameObject::setPosition(XMFLOAT3 position)
{
	m_position = position;
}

void DrawableGameObject::update(float t, const vecDrawables& drawList, const unsigned int index)
{
	XMMATRIX mSpin = XMMatrixIdentity();

	// create a list of nearby boids in the m_nearbyDrawables array
	nearbyDrawables(drawList, index);
	furtherDrawables(drawList, index);

	m_direction.x = (10 * (calculateSeparationVector(drawList).x) + (calculateCohesionVector(drawList).x) + 10*(calculateAlignmentVector(drawList).x))/21;
	m_direction.y = (10 * (calculateSeparationVector(drawList).y) + (calculateCohesionVector(drawList).y) + 10*(calculateAlignmentVector(drawList).y))/21;

	m_position.x = m_position.x + (m_direction.x) * t * OBJECT_SPEED;
	m_position.y = m_position.y + (m_direction.y) * t * OBJECT_SPEED;

	XMMATRIX mTranslate = XMMatrixTranslation(m_position.x, m_position.y, m_position.z);
	m_World = mSpin * mTranslate;
}


void DrawableGameObject::nearbyDrawables(const vecDrawables& drawList, const unsigned int index)
{
	if (m_nearbyDrawables == nullptr) {
		if (drawList.size() == 0)
			return;
		m_nearbyDrawables = new unsigned int[drawList.size()+5];
	}

	unsigned int nearbyIndex = 0;
	for (unsigned int i = 0; i < drawList.size(); i++) {
		// ignore self
		if (i == index)
			continue;

		// get the distance between the two
		XMFLOAT3 vB = *(drawList[i]->getPosition());
		XMFLOAT3 vDiff = subtractFloat3(m_position, vB);
		float l = magnitudeFloat3(vDiff);
		if (l < NEARBY_DISTANCE) {
			m_nearbyDrawables[nearbyIndex++] = i;
		}
	}

	// set the last unused element of the drawables index to UINT_MAX. Array is one larger than it needs to be
	// Self is not included
	m_nearbyDrawables[nearbyIndex] = UINT_MAX;
}

void DrawableGameObject::furtherDrawables(const vecDrawables& drawList, const unsigned int index)
{
	if (m_furtherDrawables == nullptr) {
		if (drawList.size() == 0)
			return;
		m_furtherDrawables = new unsigned int[drawList.size() + 5];
	}

	unsigned int nearbyIndex = 0;
	for (unsigned int i = 0; i < drawList.size(); i++) {
		// ignore self
		if (i == index)
			continue;

		// get the distance between the two
		XMFLOAT3 vB = *(drawList[i]->getPosition());
		XMFLOAT3 vDiff = subtractFloat3(m_position, vB);
		float l = magnitudeFloat3(vDiff);
		if (l < FURTHER_DISTANCE) {
			m_furtherDrawables[nearbyIndex++] = i;
		}
	}

	// set the last unused element of the drawables index to UINT_MAX. Note: because the array is one larger than it needs to be
	// this should always be safe even if *all* are nearby (because self is not included)
	m_furtherDrawables[nearbyIndex] = UINT_MAX;
}

void DrawableGameObject::draw(ID3D11DeviceContext* pContext)
{
	pContext->DrawIndexed(NUM_VERTICES, 0, 0);
}

void DrawableGameObject::checkIsOnScreenAndFix(const XMMATRIX&  view, const XMMATRIX&  proj)
{
	XMFLOAT4 v4;
	v4.x = m_position.x;
	v4.y = m_position.y;
	v4.z = m_position.z;
	v4.w = 1.0f;

	XMVECTOR vScreenSpace = XMLoadFloat4(&v4); 
	XMVECTOR vScreenSpace2 = XMVector4Transform(vScreenSpace, view);
	XMVECTOR vScreenSpace3 = XMVector4Transform(vScreenSpace2, proj);
	
	XMFLOAT4 v;
	XMStoreFloat4(&v, vScreenSpace3);
	v.x /= v.w;
	v.y /= v.w;
	v.z /= v.w;
	v.w /= v.w;

	// this is the position. the fish is visible on screen if the x is -1 to 1 and the same with y
	float fOffset = 40; // a suitable distance to rectify position within clip space
	if (v.x < -1 || v.x > 1 || v.y < -1 || v.y > 1)
	{
		if (v.x < -1 || v.x > 1) {
			if (v.x < -1)	//if off left screen.
			{
				m_direction.x = -10 * m_direction.x;
				m_direction = normaliseFloat3(m_direction);
				m_position.x = m_position.x + fOffset;
			}
			else if (v.x > 1)	//if off right screen.
			{
				m_direction.x = -10*m_direction.x;
				m_direction = normaliseFloat3(m_direction);
				m_position.x = m_position.x - fOffset;
			}
		}
		else if (v.y < -1 || v.y > 1) {
			if (v.y < -1)	//if off bottom screen.
			{
				m_direction.y = -10 * m_direction.y;
				m_direction = normaliseFloat3(m_direction);
				m_position.y = m_position.y + fOffset;
			}
			else if (v.y > 1)	//if off top screen.
			{
				m_direction.y = -10 * m_direction.y;
				m_direction = normaliseFloat3(m_direction);
				m_position.y = m_position.y - fOffset;
			}
		}
	}
	return;
}

XMFLOAT3 DrawableGameObject::calculateSeparationVector(const vecDrawables& drawList)
{
	XMFLOAT3 nearby = XMFLOAT3(0, 0, 0);
	if (drawList.size() == 0 || m_nearbyDrawables == nullptr)
		return m_direction;

	XMFLOAT3 totalNearbyPosition = XMFLOAT3(0, 0, 0);
	XMFLOAT3 headingVector = XMFLOAT3(0, 0, 0);
	float distance = 0.0f;
	float scale = 0.0f;

	unsigned int index = 0;
	while (m_nearbyDrawables[index] != UINT_MAX)	//UNINT_MAX is the last index.
	{
		DrawableGameObject* pFish = (drawList[m_nearbyDrawables[index]]);
		distance = sqrt(((m_position.x - pFish->getPosition()->x) * (m_position.x - pFish->getPosition()->x)) + ((m_position.y - pFish->getPosition()->y) * (m_position.y - pFish->getPosition()->y)));
		totalNearbyPosition.x = totalNearbyPosition.x + pFish->getPosition()->x;
		totalNearbyPosition.y = totalNearbyPosition.y + pFish->getPosition()->y;
		index++;
	}
	if (index == 0)
		return m_direction;

	//Create a scale for how close the agents are together.
	scale = distance / sqrt(NEARBY_DISTANCE * NEARBY_DISTANCE);
	scale = 1 - scale;

	headingVector = divideFloat3(totalNearbyPosition, index);
	headingVector = subtractFloat3(headingVector, m_position);
	headingVector = multiplyFloat3(headingVector, -1);
	headingVector = divideFloat3(headingVector, scale);
	nearby = normaliseFloat3(headingVector);

	return nearby; // average direction away from fish to nearby
}

XMFLOAT3 DrawableGameObject::calculateAlignmentVector(const vecDrawables& drawList)
{
	XMFLOAT3 nearby = XMFLOAT3(0, 0, 0);
	if (drawList.size() == 0 || m_furtherDrawables == nullptr)
		return m_direction;

	XMFLOAT3 totalNearbyDirection = XMFLOAT3(0, 0, 0);

	unsigned int index = 0;
	while (m_furtherDrawables[index] != UINT_MAX)	//UNINT_MAX is the last index.
	{
		DrawableGameObject* pFish = (drawList[m_furtherDrawables[index]]);
		totalNearbyDirection.x = totalNearbyDirection.x + pFish->getDirection()->x;
		totalNearbyDirection.y = totalNearbyDirection.y + pFish->getDirection()->y;
		index++;
	}
	if (index == 0)
		return m_direction;

	nearby = divideFloat3(totalNearbyDirection, index);		//Average direction of nearby objects
	nearby = normaliseFloat3(nearby);	//Normalise to smooth out motion

	return nearby; // return the average direction of nearby drawables
}


XMFLOAT3 DrawableGameObject::calculateCohesionVector(const vecDrawables& drawList)
{
	XMFLOAT3 all = XMFLOAT3(0, 0, 0);

	if (drawList.size() == 0 || m_furtherDrawables == nullptr)
		return m_direction;

	XMFLOAT3 positionTotal = XMFLOAT3(0, 0, 0);
	XMFLOAT3 avg = XMFLOAT3(0, 0, 0);
	unsigned int index = 0;
	while (m_furtherDrawables[index] != UINT_MAX)//(index < drawList.size())
	{
		DrawableGameObject* pFish = (drawList[m_furtherDrawables[index]]);	// (drawList[index]);
		positionTotal.x = positionTotal.x + pFish->getPosition()->x;
		positionTotal.y = positionTotal.y + pFish->getPosition()->y;
		index++;
	}
	if (index == 0)
		return m_direction;

	avg = divideFloat3(positionTotal, index);		//Average direction of further nearby objects
	avg = subtractFloat3(avg, m_position);		
	avg = normaliseFloat3(avg);		//Normalise so that position change is smooth

	return avg; // return avg direction from this fish to all
}

// --------------------------------------------------------------------------------
// Math functions

XMFLOAT3 DrawableGameObject::addFloat3(XMFLOAT3& f1, XMFLOAT3& f2)
{
	XMFLOAT3 out;
	out.x = f1.x + f2.x;
	out.y = f1.y + f2.y;
	out.z = f1.z + f2.z;

	return out;
}

XMFLOAT3 DrawableGameObject::subtractFloat3(XMFLOAT3& f1, XMFLOAT3& f2)
{
	XMFLOAT3 out;
	out.x = f1.x - f2.x;
	out.y = f1.y - f2.y;
	out.z = f1.z - f2.z;

	return out;
}

XMFLOAT3 DrawableGameObject::multiplyFloat3(XMFLOAT3& f1, const float scalar)
{
	XMFLOAT3 out;
	out.x = f1.x * scalar;
	out.y = f1.y * scalar;
	out.z = f1.z * scalar;

	return out;
}

XMFLOAT3 DrawableGameObject::divideFloat3(XMFLOAT3& f1, const float scalar)
{
	XMFLOAT3 out;
	out.x = f1.x / scalar;
	out.y = f1.y / scalar;
	out.z = f1.z / scalar;

	return out;
}

float DrawableGameObject::magnitudeFloat3(XMFLOAT3& f1)
{
	return sqrt((f1.x * f1.x) + (f1.y * f1.y) + (f1.z * f1.z));
}

XMFLOAT3 DrawableGameObject::normaliseFloat3(XMFLOAT3& f1)
{
	float length = sqrt((f1.x * f1.x) + (f1.y * f1.y) + (f1.z * f1.z));

	f1.x /= length;
	f1.y /= length;
	f1.z /= length;

	return f1;
}
